# 延迟处理Handler

在看View Measure的流程的过程中，看到了`HandlerActionQueue`类（它是hide的），对它的解释是：**在Handler没有被创建的时候，用于保存pending work，在Handler确定好的时候，可以处理这些work。**

**就是看到写法上比较有新意，在以后自己的开发中，兴许能用得上，所以记录下！**

它的代码：

```java
/*
 * Copyright (C) 2015 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package android.view;

import android.os.Handler;

import com.android.internal.util.GrowingArrayUtils;

/**
 * Class used to enqueue pending work from Views when no Handler is attached.
 *
 * @hide Exposed for test framework only.
 */
public class HandlerActionQueue {
    private HandlerAction[] mActions;
    private int mCount;

    public void post(Runnable action) {
        postDelayed(action, 0);
    }

    public void postDelayed(Runnable action, long delayMillis) {
        final HandlerAction handlerAction = new HandlerAction(action, delayMillis);

        synchronized (this) {
            if (mActions == null) {
                mActions = new HandlerAction[4];
            }
            mActions = GrowingArrayUtils.append(mActions, mCount, handlerAction);
            mCount++;
        }
    }

    public void removeCallbacks(Runnable action) {
        synchronized (this) {
            final int count = mCount;
            int j = 0;

            final HandlerAction[] actions = mActions;
            for (int i = 0; i < count; i++) {
                if (actions[i].matches(action)) {
                    // Remove this action by overwriting it within
                    // this loop or nulling it out later.
                    continue;
                }

                if (j != i) {
                    // At least one previous entry was removed, so
                    // this one needs to move to the "new" list.
                    actions[j] = actions[i];
                }

                j++;
            }

            // The "new" list only has j entries.
            mCount = j;

            // Null out any remaining entries.
            for (; j < count; j++) {
                actions[j] = null;
            }
        }
    }

    public void executeActions(Handler handler) {
        synchronized (this) {
            final HandlerAction[] actions = mActions;
            for (int i = 0, count = mCount; i < count; i++) {
                final HandlerAction handlerAction = actions[i];
                handler.postDelayed(handlerAction.action, handlerAction.delay);
            }

            mActions = null;
            mCount = 0;
        }
    }

    public int size() {
        return mCount;
    }

    public Runnable getRunnable(int index) {
        if (index >= mCount) {
            throw new IndexOutOfBoundsException();
        }
        return mActions[index].action;
    }

    public long getDelay(int index) {
        if (index >= mCount) {
            throw new IndexOutOfBoundsException();
        }
        return mActions[index].delay;
    }

    private static class HandlerAction {
        final Runnable action;
        final long delay;

        public HandlerAction(Runnable action, long delay) {
            this.action = action;
            this.delay = delay;
        }

        public boolean matches(Runnable otherAction) {
            return otherAction == null && action == null
                    || action != null && action.equals(otherAction);
        }
    }
}
```

它又个辅助类`GrowingArrayUtils`：

```java
/*
 * Copyright (C) 2015 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package android.support.v7.content.res;

import java.lang.reflect.Array;

/**
 * A helper class that aims to provide comparable growth performance to ArrayList, but on primitive
 * arrays. Common array operations are implemented for efficient use in dynamic containers.
 *
 * All methods in this class assume that the length of an array is equivalent to its capacity and
 * NOT the number of elements in the array. The current size of the array is always passed in as a
 * parameter.
 */
final class GrowingArrayUtils {

    /**
     * Appends an element to the end of the array, growing the array if there is no more room.
     * @param array The array to which to append the element. This must NOT be null.
     * @param currentSize The number of elements in the array. Must be less than or equal to
     *                    array.length.
     * @param element The element to append.
     * @return the array to which the element was appended. This may be different than the given
     *         array.
     */
    public static <T> T[] append(T[] array, int currentSize, T element) {
        assert currentSize <= array.length;

        if (currentSize + 1 > array.length) {
            T[] newArray = (T[]) Array.newInstance(array.getClass().getComponentType(),
                    growSize(currentSize));
            System.arraycopy(array, 0, newArray, 0, currentSize);
            array = newArray;
        }
        array[currentSize] = element;
        return array;
    }

    /**
     * Primitive int version of {@link #append(Object[], int, Object)}.
     */
    public static int[] append(int[] array, int currentSize, int element) {
        assert currentSize <= array.length;

        if (currentSize + 1 > array.length) {
            int[] newArray = new int[growSize(currentSize)];
            System.arraycopy(array, 0, newArray, 0, currentSize);
            array = newArray;
        }
        array[currentSize] = element;
        return array;
    }

    /**
     * Primitive long version of {@link #append(Object[], int, Object)}.
     */
    public static long[] append(long[] array, int currentSize, long element) {
        assert currentSize <= array.length;

        if (currentSize + 1 > array.length) {
            long[] newArray = new long[growSize(currentSize)];
            System.arraycopy(array, 0, newArray, 0, currentSize);
            array = newArray;
        }
        array[currentSize] = element;
        return array;
    }

    /**
     * Primitive boolean version of {@link #append(Object[], int, Object)}.
     */
    public static boolean[] append(boolean[] array, int currentSize, boolean element) {
        assert currentSize <= array.length;

        if (currentSize + 1 > array.length) {
            boolean[] newArray = new boolean[growSize(currentSize)];
            System.arraycopy(array, 0, newArray, 0, currentSize);
            array = newArray;
        }
        array[currentSize] = element;
        return array;
    }

    /**
     * Inserts an element into the array at the specified index, growing the array if there is no
     * more room.
     *
     * @param array The array to which to append the element. Must NOT be null.
     * @param currentSize The number of elements in the array. Must be less than or equal to
     *                    array.length.
     * @param element The element to insert.
     * @return the array to which the element was appended. This may be different than the given
     *         array.
     */
    public static <T> T[] insert(T[] array, int currentSize, int index, T element) {
        assert currentSize <= array.length;

        if (currentSize + 1 <= array.length) {
            System.arraycopy(array, index, array, index + 1, currentSize - index);
            array[index] = element;
            return array;
        }

        T[] newArray = (T[]) Array.newInstance(array.getClass().getComponentType(),
                growSize(currentSize));
        System.arraycopy(array, 0, newArray, 0, index);
        newArray[index] = element;
        System.arraycopy(array, index, newArray, index + 1, array.length - index);
        return newArray;
    }

    /**
     * Primitive int version of {@link #insert(Object[], int, int, Object)}.
     */
    public static int[] insert(int[] array, int currentSize, int index, int element) {
        assert currentSize <= array.length;

        if (currentSize + 1 <= array.length) {
            System.arraycopy(array, index, array, index + 1, currentSize - index);
            array[index] = element;
            return array;
        }

        int[] newArray = new int[growSize(currentSize)];
        System.arraycopy(array, 0, newArray, 0, index);
        newArray[index] = element;
        System.arraycopy(array, index, newArray, index + 1, array.length - index);
        return newArray;
    }

    /**
     * Primitive long version of {@link #insert(Object[], int, int, Object)}.
     */
    public static long[] insert(long[] array, int currentSize, int index, long element) {
        assert currentSize <= array.length;

        if (currentSize + 1 <= array.length) {
            System.arraycopy(array, index, array, index + 1, currentSize - index);
            array[index] = element;
            return array;
        }

        long[] newArray = new long[growSize(currentSize)];
        System.arraycopy(array, 0, newArray, 0, index);
        newArray[index] = element;
        System.arraycopy(array, index, newArray, index + 1, array.length - index);
        return newArray;
    }

    /**
     * Primitive boolean version of {@link #insert(Object[], int, int, Object)}.
     */
    public static boolean[] insert(boolean[] array, int currentSize, int index, boolean element) {
        assert currentSize <= array.length;

        if (currentSize + 1 <= array.length) {
            System.arraycopy(array, index, array, index + 1, currentSize - index);
            array[index] = element;
            return array;
        }

        boolean[] newArray = new boolean[growSize(currentSize)];
        System.arraycopy(array, 0, newArray, 0, index);
        newArray[index] = element;
        System.arraycopy(array, index, newArray, index + 1, array.length - index);
        return newArray;
    }

    /**
     * Given the current size of an array, returns an ideal size to which the array should grow.
     * This is typically double the given size, but should not be relied upon to do so in the
     * future.
     */
    public static int growSize(int currentSize) {
        return currentSize <= 4 ? 8 : currentSize * 2;
    }

    // Uninstantiable
    private GrowingArrayUtils() {}
}
```