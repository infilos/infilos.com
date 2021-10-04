---
type: docs
title: "CH25-Stack"
linkTitle: "CH25-Stack"
weight: 25
---

Stack和Vector一样，从jdk1.0就存在，Stack的代码相当简单，通过继承Vector类，实现了简单的Stack:

```java
/**
 * Creates an empty Stack.
 */
public Stack() {
}

/**
 * Pushes an item onto the top of this stack. This has exactly
 * the same effect as:
 * <blockquote><pre>
 * addElement(item)</pre></blockquote>
 *
 * @param   item   the item to be pushed onto this stack.
 * @return  the <code>item</code> argument.
 * @see     java.util.Vector#addElement
 */
public E push(E item) {
    addElement(item);

    return item;
}

/**
 * Removes the object at the top of this stack and returns that
 * object as the value of this function.
 *
 * @return  The object at the top of this stack (the last item
 *          of the <tt>Vector</tt> object).
 * @throws  EmptyStackException  if this stack is empty.
 */
public synchronized E pop() {
    E       obj;
    int     len = size();

    obj = peek();
    removeElementAt(len - 1);

    return obj;
}

/**
 * Looks at the object at the top of this stack without removing it
 * from the stack.
 *
 * @return  the object at the top of this stack (the last item
 *          of the <tt>Vector</tt> object).
 * @throws  EmptyStackException  if this stack is empty.
 */
public synchronized E peek() {
    int     len = size();

    if (len == 0)
        throw new EmptyStackException();
    return elementAt(len - 1);
}

/**
 * Tests if this stack is empty.
 *
 * @return  <code>true</code> if and only if this stack contains
 *          no items; <code>false</code> otherwise.
 */
public boolean empty() {
    return size() == 0;
}

/**
 * Returns the 1-based position where an object is on this stack.
 * If the object <tt>o</tt> occurs as an item in this stack, this
 * method returns the distance from the top of the stack of the
 * occurrence nearest the top of the stack; the topmost item on the
 * stack is considered to be at distance <tt>1</tt>. The <tt>equals</tt>
 * method is used to compare <tt>o</tt> to the
 * items in this stack.
 *
 * @param   o   the desired object.
 * @return  the 1-based position from the top of the stack where
 *          the object is located; the return value <code>-1</code>
 *          indicates that the object is not on the stack.
 */
public synchronized int search(Object o) {
    int i = lastIndexOf(o);

    if (i >= 0) {
        return size() - i;
    }
    return -1;
}
```

Stack也是线程安全的，但是问题是它具有Vector的缺点，由于每次push一个元素都会扩容，这导致效率比较低下。正因为如此，Stack也不推荐使用，使用LinkedList效率会更高。