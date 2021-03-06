# Spannable.Factory
```java
//android.widget.TextView

/**
  * Sets the Factory used to create new Spannables.
  */
public final void setSpannableFactory(Spannable.Factory factory) {
    mSpannableFactory = factory;
    setText(mText);
}
```
从注释可以看出这个方法是给TextView设置一个创建Spannables的工厂类。通过这个工厂类可以创建自己的SpannableString。具体创建过程如下
```java
private BufferType mBufferType = BufferType.NORMAL;

public final void setText(CharSequence text) {
    setText(text, mBufferType);
}
public void setText(CharSequence text, BufferType type) {
    setText(text, type, true, 0);

    if (mCharWrapper != null) {
        mCharWrapper.mChars = null;
    }
}
private void setText(CharSequence text, BufferType type,
                         boolean notifyBefore, int oldlen) {
    ...
    if (type == BufferType.EDITABLE || getKeyListener() != null ||
            needEditableForNotification) {
        createEditorIfNeeded();
        mEditor.forgetUndoRedo();
        Editable t = mEditableFactory.newEditable(text);
        text = t;
        setFilters(t, mFilters);
        InputMethodManager imm = InputMethodManager.peekInstance();
        if (imm != null) imm.restartInput(this);
    } else if (type == BufferType.SPANNABLE || mMovement != null) {//1.1
        text = mSpannableFactory.newSpannable(text);
    } else if (!(text instanceof CharWrapper)) {
        text = TextUtils.stringOrSpannedString(text);
    }
    ...
    mBufferType = type;//设置新的bufferType
    mText = text;
    ...
}

public final void setMovementMethod(MovementMethod movement) {
    if (mMovement != movement) {
        mMovement = movement;

        if (movement != null && !(mText instanceof Spannable)) {
            setText(mText);
        }

        fixFocusableAndClickableSettings();

        // SelectionModifierCursorController depends on textCanBeSelected, which depends on
        // mMovement
        if (mEditor != null) mEditor.prepareCursorControllers();
    }
}
```
**1.1**

可以看到，如果type为BufferType.SPANNABLE或者mMovement不为null，就会调用mSpannableFactory创建一个新的Spannable。
通过查看TextView的源码发现mBufferType这个默认情况下是BufferType.NORMAL。只有在调用setText方法之后才会从新为mBufferType赋值。

那这个新的bufferType是从哪里来的呢？通过查看setText方法的调用关系可以发现，
在调用带有两个参数的setText方法的时候，需要传入的一个BufferType参数。当我们通过调用1个参数的setText方法设置TextView的内容时，传递的就是TextView的默认的BufferType（NORMAL）。

> } else if (type == BufferType.SPANNABLE || mMovement != null) {//1.1

通过前面的分析可以知道如果想让这个条件成立即通过Spannable.Factory创建Spannable，必须调用带有两个参数的setText方法并且BufferType参数设置为BufferType.SPANNABLE或者调用setMovementMethod方法设置MovementMethod。

# Editable.Factory
```java
/**
    * Sets the Factory used to create new Editables.
    */
public final void setEditableFactory(Editable.Factory factory) {
    mEditableFactory = factory;
    setText(mText);
}

if (type == BufferType.EDITABLE || getKeyListener() != null ||
                needEditableForNotification) {//原理同Spannable.Factory
    createEditorIfNeeded();
    mEditor.forgetUndoRedo();
    Editable t = mEditableFactory.newEditable(text);
    text = t;
    setFilters(t, mFilters);
    InputMethodManager imm = InputMethodManager.peekInstance();
    if (imm != null) imm.restartInput(this);
} else if (type == BufferType.SPANNABLE || mMovement != null) {
    text = mSpannableFactory.newSpannable(text);
} else if (!(text instanceof CharWrapper)) {
    text = TextUtils.stringOrSpannedString(text);
}

```

```java
public class EditText extends TextView {
    ...
    @Override
    public void setText(CharSequence text, BufferType type) {
        super.setText(text, BufferType.EDITABLE);
    }
    ...
}
```
EditText重写了两个参数的setText方法，并且参数BufferType固定传BufferType.EDITABLE

***
# SpannableString

* SpannableString//内容不可变，样式可变
* SpannedString//内容和样式都不可变
* SpannableStringBuilder//内容和样式都可变

```java
public class SpannableString extends SpannableStringInternal implements CharSequence, GetChars, Spannable{

}

abstract class SpannableStringInternal{
    SpannableStringInternal(CharSequence source,
                                          int start, int end) {
        //初始化mSpans和mSpanData
        mSpans = EmptyArray.OBJECT;
        mSpanData = EmptyArray.INT;
        //如果source是Spanned，从source中获取到所有的Span，然后添加进来
        if (source instanceof Spanned) {
            Spanned sp = (Spanned) source;
            Object[] spans = sp.getSpans(start, end, Object.class);

            for (int i = 0; i < spans.length; i++) {
                int st = sp.getSpanStart(spans[i]);
                int en = sp.getSpanEnd(spans[i]);
                int fl = sp.getSpanFlags(spans[i]);

                if (st < start)
                    st = start;
                if (en > end)
                    en = end;

                setSpan(spans[i], st - start, en - start, fl);
            }
        }
    }

    void setSpan(Object what, int start, int end, int flags) {
        ...
        int count = mSpanCount;
        Object[] spans = mSpans;
        int[] data = mSpanData;
        //如果新设置的样式what存在，更新样式的起始位置和flag
        for (int i = 0; i < count; i++) {
            if (spans[i] == what) {
                int ostart = data[i * COLUMNS + START];
                int oend = data[i * COLUMNS + END];

                data[i * COLUMNS + START] = start;
                data[i * COLUMNS + END] = end;
                data[i * COLUMNS + FLAGS] = flags;

                sendSpanChanged(what, ostart, oend, nstart, nend);
                return;
            }
        }

        //如果长度不够，扩容mSpans数组
        if (mSpanCount + 1 >= mSpans.length) {
            Object[] newtags = ArrayUtils.newUnpaddedObjectArray(
                    GrowingArrayUtils.growSize(mSpanCount));
            int[] newdata = new int[newtags.length * 3];

            System.arraycopy(mSpans, 0, newtags, 0, mSpanCount);
            System.arraycopy(mSpanData, 0, newdata, 0, mSpanCount * 3);

            mSpans = newtags;
            mSpanData = newdata;
        }

        //！！！！！！！重点
        //把样式what存放到mSpans数组中
        mSpans[mSpanCount] = what;
        //mSpanData数组存放了样式的3要素（start、end、flags），每一个要素占用mSpanData数组的一个位置。所以通过mSpanCount * COLUMNS计算出要素在mSpanCount数组的开始位置，然后在加上START、END、FLAGS就可以算出元素的具体下标。
        mSpanData[mSpanCount * COLUMNS + START] = start;
        mSpanData[mSpanCount * COLUMNS + END] = end;
        mSpanData[mSpanCount * COLUMNS + FLAGS] = flags;
        mSpanCount++;
    }
    private String mText;//要显示的字符串
    private Object[] mSpans;//所有样式
    private int[] mSpanData;//样式的信息（开始位置start，结束位置end,flag）
    private int mSpanCount;//样式的数量

    /* package */ static final Object[] EMPTY = new Object[0];

    private static final int START = 0;
    private static final int END = 1;
    private static final int FLAGS = 2;
    private static final int COLUMNS = 3;//对应样式的3要素（start/end/flags）
}
```

