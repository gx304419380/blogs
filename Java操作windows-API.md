使用jna自带的功能进行调用
maven依赖如下：
```
        <dependency>
            <groupId>net.java.dev.jna</groupId>
            <artifactId>jna</artifactId>
            <version>5.4.0</version>
        </dependency>
        <dependency>
            <groupId>net.java.dev.jna</groupId>
            <artifactId>jna-platform</artifactId>
            <version>5.4.0</version>
        </dependency>
```


代码如下：
```
package com.fly.jna.window.util;

import com.sun.jna.Memory;
import com.sun.jna.Pointer;
import com.sun.jna.platform.win32.Kernel32;
import com.sun.jna.platform.win32.User32;
import com.sun.jna.platform.win32.WinDef.HWND;
import com.sun.jna.platform.win32.WinNT;
import com.sun.jna.platform.win32.WinNT.HANDLE;
import com.sun.jna.ptr.IntByReference;

public class WindowsUtils {

    private static final Kernel32 KERNEL = Kernel32.INSTANCE;

    private static final User32 USER = User32.INSTANCE;


    /**
     * 通过窗口名称获取句柄
     * @param windowName    窗口名称
     * @return  句柄
     */
    public static HANDLE getHandleByWindowName(String windowName) {
        HWND hwnd = USER.FindWindow(null, windowName);
        IntByReference pidReference = new IntByReference();
        USER.GetWindowThreadProcessId(hwnd, pidReference);
        int pid = pidReference.getValue();

        return KERNEL.OpenProcess(WinNT.PROCESS_ALL_ACCESS, false, pid);
    }


    /**
     * 通过pid获取句柄
     * @param pid    pid
     * @return  句柄
     */
    public static HANDLE getHandleByPid(int pid) {
        return KERNEL.OpenProcess(WinNT.PROCESS_ALL_ACCESS, false, pid);
    }


    /**
     * 将int值写入到指定地址
     * @param handle    程序句柄
     * @param address   地址
     * @param value     值
     * @return  写入多少 -1为失败
     */
    public static int writeIntToAddress(HANDLE handle, int address, int value) {
        Memory memory = new Memory(4);
        memory.setInt(0, value);
        IntByReference readByteNumber = new IntByReference();
        KERNEL.WriteProcessMemory(handle, Pointer.createConstant(address), memory, 4, readByteNumber);
        return readByteNumber.getValue();
    }

    /**
     * 读取指定地址的int值
     * @param handle    程序句柄
     * @param address   地址
     * @return  值
     */
    public static int readIntOfAddress(HANDLE handle, int address) {
        Memory memory = new Memory(4);
        IntByReference readByteNumber = new IntByReference();
        KERNEL.ReadProcessMemory(handle, Pointer.createConstant(address), memory, 4, readByteNumber);
        return memory.getInt(0);
    }


    /**
     * 读取指定地址的int值
     * @param handle    程序句柄
     * @param pointer   地址
     * @return  值
     */
    public static int readIntOfPointer(HANDLE handle, Pointer pointer) {
        Memory memory = new Memory(4);
        IntByReference readByteNumber = new IntByReference();
        KERNEL.ReadProcessMemory(handle, pointer, memory, 4, readByteNumber);
        return memory.getInt(0);
    }

    /**
     * 读取指定地址的int值
     * @param pointer   地址
     * @return  值
     */
    public static int readIntOfExPointer(ExPointer pointer) {
        Memory memory = new Memory(4);
        IntByReference readByteNumber = new IntByReference();
        KERNEL.ReadProcessMemory(pointer.getHandle(), pointer.getPointer(), memory, 4, readByteNumber);
        return memory.getInt(0);
    }


    /**
     * 读取指定地址的double值
     * @param exPointer   地址
     * @return  值
     */
    public static double readDoubleOfExPointer(ExPointer exPointer) {
        Memory memory = new Memory(8);
        IntByReference readByteNumber = new IntByReference();
        KERNEL.ReadProcessMemory(exPointer.getHandle(), exPointer.getPointer(), memory, 8, readByteNumber);
        return memory.getDouble(0);
    }


    /**
     * 读取指定地址的float值
     * @param exPointer   地址
     * @return  值
     */
    public static float readFloatOfExPointer(ExPointer exPointer) {
        Memory memory = new Memory(8);
        IntByReference readByteNumber = new IntByReference();
        KERNEL.ReadProcessMemory(exPointer.getHandle(), exPointer.getPointer(), memory, 8, readByteNumber);
        return memory.getFloat(0);
    }

    /**
     * 读取指定地址的float值
     * @param exPointer   地址
     * @return  值
     */
    public static long readLongOfExPointer(ExPointer exPointer) {
        Memory memory = new Memory(8);
        IntByReference readByteNumber = new IntByReference();
        KERNEL.ReadProcessMemory(exPointer.getHandle(), exPointer.getPointer(), memory, 8, readByteNumber);
        return memory.getLong(0);
    }

    /**
     * 读取指定地址的float值
     * @param exPointer   地址
     * @return  值
     */
    public static byte readByteOfExPointer(ExPointer exPointer) {
        Memory memory = new Memory(1);
        IntByReference readByteNumber = new IntByReference();
        KERNEL.ReadProcessMemory(exPointer.getHandle(), exPointer.getPointer(), memory, 1, readByteNumber);
        return memory.getByte(0);
    }

    /**
     * 读取指定地址的float值
     * @param exPointer   地址
     * @return  值
     */
    public static byte[] readByteArray(ExPointer exPointer, int size) {
        Memory memory = new Memory(size);
        IntByReference readByteNumber = new IntByReference();
        KERNEL.ReadProcessMemory(exPointer.getHandle(), exPointer.getPointer(), memory, size, readByteNumber);
        return memory.getByteArray(0, size);
    }

    /**
     * 写入到地址
     * @param exPointer 指针
     * @param value     值
     */
    public static void writeValueToExPointer(ExPointer exPointer, Object value) {
        Memory memory = null;
        if (value instanceof Integer) {
            memory = new Memory(4);
            memory.setInt(0, (Integer) value);
        } else if (value instanceof Float) {
            memory = new Memory(4);
            memory.setFloat(0, (Float) value);
        } else if (value instanceof Double) {
            memory = new Memory(8);
            memory.setDouble(0, (Double) value);
        } else  if (value instanceof Long) {
            memory = new Memory(8);
        } else if (value instanceof byte[]) {
            byte[] bytes = (byte[]) value;
            memory = new Memory(bytes.length);
            memory.write(0, bytes, 0, bytes.length);
        }

        IntByReference readByteNumber = new IntByReference();
        KERNEL.WriteProcessMemory(exPointer.getHandle(), exPointer.getPointer(), memory, 4, readByteNumber);
    }
}

```

一个自定义的指针类
```
package com.fly.jna.window.util;

import com.sun.jna.Pointer;
import com.sun.jna.platform.win32.WinNT.HANDLE;
import lombok.Data;
import lombok.experimental.Accessors;

@Data
@Accessors(chain = true)
public class ExPointer {

    private Pointer pointer;
    private HANDLE handle;

    public ExPointer() {
    }

    public ExPointer(HANDLE handle, int address) {
        this.handle = handle;
        pointer = Pointer.createConstant(address);
    }

    public ExPointer(Pointer pointer, HANDLE handle) {
        this.pointer = pointer;
        this.handle = handle;
    }

    public static ExPointer of(HANDLE handle, int address) {
        return new ExPointer(handle, address);
    }

    public ExPointer offset(int offset) {
        int value = WindowsUtils.readIntOfExPointer(this);
        int newAddress = value + offset;
        return new ExPointer(handle, newAddress);
    }


    /**
     * 获取指针的地址
     * @return  地址
     */
    public int getAddress() {
        return (int) Pointer.nativeValue(pointer);
    }

    /**
     * 读取指针指向的值
     * @return  int
     */
    public int readInt() {
        return WindowsUtils.readIntOfExPointer(this);
    }

    public void writeInt(int value) {
        WindowsUtils.writeValueToExPointer(this, value);
    }


    public double readDouble() {
        return WindowsUtils.readDoubleOfExPointer(this);
    }

}
```
