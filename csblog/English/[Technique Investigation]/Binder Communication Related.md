## **Topic: How to Handle Calls When Binder Connection is Broken?**

### **Further Reading**

[Principles of Android Cross-Process Exception Handling](https://cloud.tencent.com/developer/article/1741320)

### **Scenario 1: Service Process is Killed During Method Execution**

- **Result**:
  The client triggers a `DeadObjectException`. If the client does not handle this exception, it may lead to a fatal crash, causing the invocation to be lost.
- **Countermeasure**:
  The client should implement exception handling to ensure it does not crash.

------

### **Scenario 2: Service Process Throws an Exception Unsupported by AIDL During Method Execution**

- **Result**:
  The AIDL method returns default values on the client side. The client does not receive the exception, the service process does not crash, and the Binder connection remains intact.
- **Countermeasure**:
  Avoid throwing exception types in the Binder that cannot be processed across processes.

------

### **Scenario 3: Service Process Throws an AIDL-Supported Exception During Method Execution**

- **Result**:
  The exception is forwarded to the client and can be caught in the method call. The invocation is lost, but the service process does not crash. If the client does not shut down, the Binder connection remains intact.
- **Countermeasure**:
  Avoid throwing exception types in the Binder that cannot be processed across processes.

------

### **Scenario 4: Binder Connection is Broken Before Method Execution**

- **Result**:
  The client triggers a `DeadObjectException`. If unhandled, it may lead to a fatal crash, causing the invocation to be lost.
- **Countermeasure**:
  In the `onServiceDisconnected` callback, set the Binder reference to null. All Binder calls must first check if the Binder is null.

------

## **Topic: Only Nine Exception Types are Supported in Binder Inter-Process Communication**

1. **In inter-process communication, only the following nine types of exceptions can be transferred between processes**:
   - `SecurityException`
   - `BadParcelableException`
   - `IllegalArgumentException`
   - `NullPointerException`
   - `IllegalStateException`
   - `NetworkOnMainThreadException`
   - `UnsupportedOperationException`
   - `ServiceSpecificException`
   - Parcelable exceptions
2. **For unsupported exceptions, they are handled internally in the program. This may cause crashes, but they will not be passed to the other process. Examples of unsupported exceptions include**:
   - Runtime exceptions: `RuntimeException`
   - Arithmetic exceptions: `ArithmeticException`
   - Type casting exceptions: `ClassCastException`
   - Array index out-of-bounds exceptions: `ArrayIndexOutOfBoundsException`
   - File not found exceptions: `FileNotFoundException`
   - String-to-number conversion exceptions: `NumberFormatException`
   - Input/output exceptions: `IOException`
   - Method not found exceptions: `NoSuchMethodException`

------

## **Topic: Why Does Binder Only Require a Single Data Copy?**

In Linux, processes are isolated and cannot communicate directly. When process A sends data to process B, the data is first copied from the user space of A to the kernel space and then from the kernel space to the user space of B.

![img](Binder Communication Related_imgs\WMc6qL7vt9p.png)

Binder communication uses the `mmap()` function to create a memory mapping between the kernel space and the receiving user space's data buffer. This allows process A to copy data directly to the memory buffer mapped to process B's user space, effectively achieving a single data copy.

Additionally, as Binder communication is bidirectional, a similar memory mapping is created for the data buffer of process A in the kernel space.

![image-2024-9-20_17-57-45](Binder Communication Related_imgs\SWTymqDx5gh.png)