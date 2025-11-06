# What is hoisting? How do var, let, and const differ during hoisting?

## Question
What is hoisting? How do var, let, and const differ during hoisting?

## Answer

**Hoisting** is JavaScript's behavior of moving variable and function declarations to the top of their containing scope during the compilation phase, before code execution.

### How it works:

**1. var - Hoisted and initialized with undefined:**
```javascript
console.log(x); // undefined (not error!)
var x = 5;
console.log(x); // 5

// Internally treated as:
// var x = undefined;
// console.log(x);
// x = 5;
```

**2. let and const - Hoisted but NOT initialized (Temporal Dead Zone):**
```javascript
console.log(y); // ReferenceError: Cannot access 'y' before initialization
let y = 10;

console.log(z); // ReferenceError: Cannot access 'z' before initialization  
const z = 20;
```

### Key Differences Table:

| Feature | var | let | const |
|---------|-----|-----|-------|
| Hoisted? | ✅ Yes | ✅ Yes | ✅ Yes |
| Initialized during hoisting? | ✅ Yes (undefined) | ❌ No | ❌ No |
| Temporal Dead Zone? | ❌ No | ✅ Yes | ✅ Yes |
| Can access before declaration? | ✅ Yes (undefined) | ❌ No (ReferenceError) | ❌ No (ReferenceError) |

### Function Hoisting Example:
```javascript
sayHello(); // "Hello!" - works because function declarations are fully hoisted

function sayHello() {
    console.log("Hello!");
}

// But function expressions behave like variables:
sayBye(); // TypeError: sayBye is not a function
var sayBye = function() {
    console.log("Bye!");
};
```

### Interview Tip:
*"Hoisting helps avoid reference errors for functions, but with let/const, we get safer code by preventing accidental usage before initialization through the Temporal Dead Zone."*