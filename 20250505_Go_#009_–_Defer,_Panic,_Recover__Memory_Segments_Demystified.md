### **1. `defer`: Cleanup with Guaranteed Execution**  
**What it does**: Runs a function *after* its parent function finishes.  

```go  
package main  

import "os"  

func main() {  
    file, _ := os.Open("data.txt")  
    defer file.Close()  // Deferred cleanup  
}  
```  

#### **Memory Segmentation**  
```  
+-------------------+   Lowest Address  
|      Code         |  
|-------------------|  
| main()            |  // Compiled instructions  
| os.Open()         |  // File-opening logic  
| file.Close()      |  // Cleanup instructions  
+-------------------+  
|      Data         |  
|-------------------|  
| "data.txt"        |  // Filename (immutable)  
+-------------------+  
|      Stack        |  
|-------------------|  
| main() frame      |  
|   file (pointer) →|  // Points to OS resource  
+-------------------+  
|      Heap         |  
|-------------------|  
| (Empty)           |  // No allocations here  
+-------------------+   Highest Address  
```  

**Key Points**:  
- `defer` adds `file.Close()` to a LIFO stack of functions to run *after* `main()`.  
- `file` is a pointer to an OS resource (handled outside Go’s memory).  

---

### **2. `panic` & `recover`: Crash and Rescue**  
```go  
package main  

func main() {  
    defer func() {  
        if r := recover(); r != nil {  
            println("Recovered:", r)  
        }  
    }()  
    panic("critical failure")  
}  
```  

#### **Memory Segmentation**  
```  
+-------------------+   Lowest Address  
|      Code         |  
|-------------------|  
| main()            |  
| recover() logic   |  
+-------------------+  
|      Data         |  
|-------------------|  
| "critical failure"|  // Panic message  
+-------------------+  
|      Stack        |  
|-------------------|  
| main() frame      |  
|   recover defer   |  
+-------------------+  
|      Heap         |  
|-------------------|  
| (Empty)           |  // No heap allocations  
+-------------------+   Highest Address  
```  

**Key Points**:  
- `panic` stores its message in the **data segment** (immutable).  
- `recover()` checks if a panic occurred. Works **only inside deferred functions**.  

---

### **3. Heap Escape in Action**  
```go  
func main() {  
    x := 42  
    defer func() {  
        println(x)  // x escapes to heap  
    }()  
    x = 100  
}  
```  

#### **Memory Segmentation**  
```  
+-------------------+   Lowest Address  
|      Code         |  
|-------------------|  
| main()            |  
| closure logic     |  
+-------------------+  
|      Data         |  
|-------------------|  
| (No constants)    |  
+-------------------+  
|      Stack        |  
|-------------------|  
| main() frame      |  
|   x (int) →       |  // Points to heap  
+-------------------+  
|      Heap         |  
|-------------------|  
| x (int = 100)     |  // Captured by closure  
+-------------------+   Highest Address  
```  

**Why?**:  
- The deferred closure outlives `main()`, forcing `x` to the heap.  
- Verify with `go build -gcflags="-m" main.go`:  
  ```  
  ./main.go:4:3: ... x escapes to heap  
  ```  

---

### **Rules to Remember**  
1. **`defer`**  
   - Use for cleanup (files, network connections).  
   - Arguments are evaluated immediately.  

2. **`panic`**  
   - Crashes the program unless `recover()` is used.  
   - Stores messages in the **data segment**.  

3. **`recover()`**  
   - Only works inside **deferred functions**.  
   - Resets program state after a panic.  
