# What is Hoisting?

## Simple Answer
Hoisting moves variable and function declarations to the top of their scope before code runs.

## var vs let/const

**var - Hoisted and initialized as undefined:**
```javascript
console.log(x); // undefined
var x = 5;
```

**let/const - Hoisted but NOT initialized (Temporal Dead Zone):**
```javascript
console.log(y); // ReferenceError
let y = 5;
```

## Function Hoisting
```javascript
sayHello(); // Works!

function sayHello() {
  console.log("Hello!");
}
```

## Interview Q&A

**Q: What is hoisting?**  
**A:** Hoisting moves declarations to the top. var variables become undefined, let/const cause ReferenceError if accessed early.

**Q: What's the Temporal Dead Zone?**  
**A:** The time between hoisting and initialization where let/const variables can't be accessed.

**Q: Why is let/const safer than var?**  
**A:** They prevent accidental use of undefined variables and catch bugs early.