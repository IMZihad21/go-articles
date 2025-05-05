Go’s primitive types seem straightforward—until you realize how deliberately they’re designed to balance performance and memory. Let’s dissect their exact memory usage and where they live (stack, data, or heap).  

---

### **The Primitive Types Cheat Sheet**  
| Type      | Size (bytes) | Zero Value | Memory Segment  |  
|-----------|--------------|------------|-----------------|  
| `bool`    | 1            | `false`    | Stack/Data      |  
| `int32`   | 4            | 0          | Stack           |  
| `int64`   | 8            | 0          | Stack           |  
| `float32` | 4            | 0.0        | Stack           |  
| `float64` | 8            | 0.0        | Stack           |  
| `string`  | 16*          | `""`       | Data (content)  |  
| `rune`    | 4            | 0          | Stack           |  

*Strings are 16 bytes (on 64-bit machines): 8 bytes for pointer + 8 bytes for length.  

---

### **Memory Visualization**  
```go  
package main  

const Version = "1.0"  // Data segment  

func main() {  
    var (  
        isReady bool    // 1 byte (stack)  
        width   int32   // 4 bytes (stack)  
        pi      float64 // 8 bytes (stack)  
        title   string  // 16 bytes (stack), points to data  
    )  
    title = "Memory Secrets"  
}  
```  

```  
+-------------------+  
|      Data         |  
|-------------------|  
| "1.0"             |  // Version constant  
| "Memory Secrets"  |  // title's string content  
+-------------------+  
|      Stack        |  
|-------------------|  
| isReady (1 byte)  |  
| width (4 bytes)   |  
| pi (8 bytes)      |  
| title (16 bytes) →|  
+-------------------+  
|      Heap         |  
|-------------------|  
| (empty)           |  // No escapes here  
+-------------------+  
```  

---

### **Key Insights**  
#### 1. **`bool` Isn’t 1 Bit**  
Go uses 1 **byte** for `bool` (not 1 bit) because memory is addressable at the byte level. This avoids bitwise gymnastics for marginal memory savings.  

#### 2. **`string` is a Cheap Illusion**  
- The `string` variable (`title`) is 16 bytes on the stack (pointer + length).  
- The actual content (`"Memory Secrets"`) lives in the **data segment** (immutable).  

#### 3. **Zero Values = Pre-Allocated Memory**  
```go  
var count int32  // 4 bytes reserved immediately (no lazy allocation)  
```  
This prevents “garbage values” and ensures safe memory access.  

---

### **When Primitives Escape to the Heap**  
Primitives *usually* stay on the stack—unless:  
- They’re part of a **closure**.  
- You take their **address** and pass it across function boundaries.  

**Example**:  
```go  
func leak() *float64 {  
    speed := 42.0  // Escapes to heap (returned pointer)  
    return &speed  
}  
```  

Verify with `-gcflags="-m"`:  
```  
./main.go:4:2: moved to heap: speed  
```  

---

### **Why This Matters**  
- **Cache efficiency**: Stack-allocated primitives (tightly packed) leverage CPU cache better.  
- **Memory fragmentation**: Heap-allocated primitives (even tiny ones) add GC overhead.  
- **Micro-optimizations**: Use `int32` over `int` in large slices to save 4GB on 1B entries.  

---

### **The Padding Surprise**  
Go adds padding to align struct fields for CPU efficiency:  
```go  
type Data struct {  
    A bool    // 1 byte  
    B int64   // 8 bytes  
}  
// Total size: 16 bytes (1 + 7 padding + 8)  
```  
Use `unsafe.Sizeof(Data{})` to debug.  

---

**Fun Fact**: A `struct{}` (empty struct) consumes 0 bytes. Go uses it for signaling (e.g., `chan struct{}`).