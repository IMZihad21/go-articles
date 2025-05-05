Variable scope and shadowing in Go seem simple until they quietly break your logic. Let’s dissect common pitfalls with clear examples and memory visuals.

### **1. Variable Scope Basics**  
```go  
package main  

var global = "I'm global"  // Data segment  

func main() {  
    local := "I'm local"   // Stack  
    {  
        inner := "I'm inner"  // Stack (inner block)  
    }  
}  
```  

#### **Memory Segmentation**  
```  
+-------------------+   Lowest Address  
|      Code         |  
|-------------------|  
| main()            |  // Compiled instructions  
| package init      |  // Global var initialization  
+-------------------+  
|      Data         |  
|-------------------|  
| global (string)   |  // "I'm global"  
+-------------------+  
|      Stack        |  
|-------------------|  
| main() frame      |  
|   local (string)  |  
|   inner (string)  |  // Dies at inner block  
+-------------------+  
|      Heap         |  
|-------------------|  
| (empty)           |  // No allocations here  
+-------------------+   Highest Address  
```  

---

### **2. Shadowing with Global/Local Conflict**  
```go  
var count = 100  // Data segment  

func main() {  
    count := 50  // Shadows global (stack)  
    println(count)  // 50  
}  
```  

#### **Memory Segmentation**  
```  
+-------------------+  
|      Code         |  
|-------------------|  
| main()            |  // Println logic  
+-------------------+  
|      Data         |  
|-------------------|  
| count (int=100)   |  
+-------------------+  
|      Stack        |  
|-------------------|  
| main() frame      |  
|   count (int=50)  |  
+-------------------+  
|      Heap         |  
|-------------------|  
| (empty)           |  
+-------------------+  
```  

---

### **3. Closures and Heap Escape**  
```go  
func main() {  
    x := 10  
    defer func() { println(x) }()  // Closure captures x  
    x = 20  
}  
```  

#### **Memory Segmentation**  
```  
+-------------------+  
|      Code         |  
|-------------------|  
| main()            |  // defer logic  
| closure code      |  // println wrapper  
+-------------------+  
|      Data         |  
|-------------------|  
| (no constants)    |  
+-------------------+  
|      Stack        |  
|-------------------|  
| main() frame      |  
|   x (pointer) →   |  // Points to heap  
+-------------------+  
|      Heap         |  
|-------------------|  
| x (int=20)        |  // Captured by closure  
+-------------------+  
```  

**Why?**: The closure outlives `main()`, forcing `x` to the heap.  

---

### **4. Loop Variable Shadowing Fix**  
```go  
func main() {  
    var funcs []func()  
    for i := 0; i < 3; i++ {  
        i := i  // New stack variable per iteration  
        funcs = append(funcs, func() { println(i) })  
    }  
}  
```  

#### **Memory Segmentation**  
```  
+-------------------+  
|      Code         |  
|-------------------|  
| main()            |  // Loop logic  
| closure code      |  // println wrappers  
+-------------------+  
|      Data         |  
|-------------------|  
| (no constants)    |  
+-------------------+  
|      Stack        |  
|-------------------|  
| main() frame      |  
|   i (int=0,1,2)  |  // One per iteration  
+-------------------+  
|      Heap         |  
|-------------------|  
| closures          |  // Each captures its own i  
+-------------------+  
```  

---

### **Rules to Remember**  
1. **Code Segment**: Holds compiled instructions. Always present.  
2. **Data Segment**: Global variables, constants.  
3. **Stack**: Local variables (function/block scope).  
4. **Heap**: Variables captured by closures or explicitly allocated.  
