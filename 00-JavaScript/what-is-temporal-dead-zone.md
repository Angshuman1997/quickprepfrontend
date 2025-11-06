# What is the Temporal Dead Zone?

## Question
What is the Temporal Dead Zone?

## Answer

**Temporal Dead Zone (TDZ)** is the period between the start of a scope and the point where a `let` or `const` variable is declared and initialized. During this time, the variable exists but cannot be accessed.

### Visual Example:
```javascript
console.log("Start of scope");

// TDZ starts here for 'myVariable'
console.log(myVariable); // ReferenceError: Cannot access 'myVariable' before initialization

let myVariable = "Hello"; // TDZ ends here
console.log(myVariable); // "Hello" - now accessible
```

### TDZ with Different Variable Types:

**1. let variables:**
```javascript
function example() {
    // TDZ for 'a' starts here
    console.log(a); // ReferenceError
    
    let a = 10; // TDZ for 'a' ends here
    console.log(a); // 10
}
```

**2. const variables:**
```javascript
function example() {
    // TDZ for 'b' starts here
    console.log(b); // ReferenceError
    
    const b = 20; // TDZ for 'b' ends here
    console.log(b); // 20
}
```

**3. var has NO TDZ:**
```javascript
function example() {
    console.log(c); // undefined (not ReferenceError)
    var c = 30;
    console.log(c); // 30
}
```

### Real-world Example:
```javascript
function processOrder(items) {
    if (items.length > 0) {
        // TDZ starts here for 'total'
        
        for (let item of items) {
            // Can't use 'total' here yet!
            // console.log(total); // ReferenceError
        }
        
        let total = calculateTotal(items); // TDZ ends here
        console.log(total); // Now accessible
    }
}
```

### Why TDZ exists:
- **Prevents bugs** from using variables before initialization
- **Encourages better code** by forcing proper declaration order
- **Makes debugging easier** by throwing clear errors

### Interview Tip:
*"TDZ helps catch common bugs early. It's JavaScript's way of saying 'declare before you use' - making our code more predictable and safer."*