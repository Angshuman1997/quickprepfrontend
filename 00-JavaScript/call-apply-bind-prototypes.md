# Explain call, apply, and bind methods. What are prototypes?

## Question
Explain call, apply, and bind methods. What are prototypes?

## Answer

## Part 1: call, apply, and bind Methods

**`call`**, **`apply`**, and **`bind`** are methods available on all JavaScript functions that allow you to control the `this` context and pass arguments in different ways. They are essential for function borrowing, method sharing, and controlling execution context.

### 1. **`call()` Method**

**Purpose**: Calls a function with a given `this` value and individual arguments.

**Syntax**: `function.call(thisArg, arg1, arg2, ...)`

#### Basic Example:
```javascript
function greet(message) {
    return `${message}, ${this.name}!`;
}

const person = { name: 'Alice' };

// Using call to set 'this' to person
console.log(greet.call(person, 'Hello')); // "Hello, Alice!"
```

#### Function Borrowing Example:
```javascript
const person1 = { name: 'John', age: 30 };
const person2 = { name: 'Jane', age: 25 };

function introduce() {
    return `Hi, I'm ${this.name} and I'm ${this.age} years old.`;
}

// Borrow the introduce function for different objects
console.log(introduce.call(person1)); // "Hi, I'm John and I'm 30 years old."
console.log(introduce.call(person2)); // "Hi, I'm Jane and I'm 25 years old."
```

### 2. **`apply()` Method**

**Purpose**: Similar to `call()`, but takes arguments as an array instead of individual parameters.

**Syntax**: `function.apply(thisArg, [argsArray])`

#### Finding Max/Min in Array:
```javascript
const numbers = [5, 2, 8, 1, 9, 3];

// Using apply to spread array as arguments
const max = Math.max.apply(null, numbers); // 9
const min = Math.min.apply(null, numbers); // 1

// Modern way (ES6 spread operator)
const maxModern = Math.max(...numbers); // 9
```

### 3. **`bind()` Method**

**Purpose**: Creates a new function with a permanently bound `this` value, but doesn't execute it immediately.

**Syntax**: `function.bind(thisArg, arg1, arg2, ...)`

#### Partial Application:
```javascript
function multiply(a, b) {
    return a * b;
}

// Create a function that always multiplies by 2
const double = multiply.bind(null, 2);

console.log(double(5));  // 10
console.log(double(10)); // 20
```

#### Key Differences:

| Method | Immediate Execution | Arguments | Returns |
|--------|-------------------|------------|---------|
| `call()` | ✅ Yes | Individual arguments | Function result |
| `apply()` | ✅ Yes | Array of arguments | Function result |
| `bind()` | ❌ No | Individual arguments | New bound function |

## Part 2: Prototypes in JavaScript

**Prototypes** are the mechanism by which JavaScript objects inherit properties and methods from other objects. Every JavaScript object has a prototype, and when you access a property that doesn't exist on the object itself, JavaScript looks up the prototype chain.

### Prototype Chain Basics:

```javascript
// Constructor function
function Person(name, age) {
    this.name = name;
    this.age = age;
}

// Adding method to prototype
Person.prototype.greet = function() {
    return `Hello, I'm ${this.name}`;
};

// Creating instances
const alice = new Person('Alice', 30);
const bob = new Person('Bob', 25);

console.log(alice.greet()); // "Hello, I'm Alice"
console.log(bob.greet());  // "Hello, I'm Bob"

// Both share the same prototype method
console.log(alice.greet === bob.greet); // true
```

### Prototype vs Instance Properties:

```javascript
function Car(make, model) {
    this.make = make;        // Instance property
    this.model = model;      // Instance property
    this.mileage = 0;        // Instance property
}

// Prototype properties (shared)
Car.prototype.drive = function(miles) {
    this.mileage += miles;
    return `${this.make} ${this.model} has driven ${this.mileage} miles`;
};

Car.prototype.getInfo = function() {
    return `${this.make} ${this.model} (${this.mileage} miles)`;
};

const car1 = new Car('Toyota', 'Camry');
const car2 = new Car('Honda', 'Civic');

car1.drive(100);
car2.drive(50);

console.log(car1.getInfo()); // "Toyota Camry (100 miles)"
console.log(car2.getInfo()); // "Honda Civic (50 miles)"
```

### Prototype Inheritance:

```javascript
// Parent constructor
function Animal(name) {
    this.name = name;
}

Animal.prototype.eat = function() {
    return `${this.name} is eating`;
};

Animal.prototype.sleep = function() {
    return `${this.name} is sleeping`;
};

// Child constructor
function Dog(name, breed) {
    Animal.call(this, name); // Call parent constructor
    this.breed = breed;
}

// Set up inheritance
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;

// Add dog-specific methods
Dog.prototype.bark = function() {
    return `${this.name} says woof!`;
};

// Override parent method
Dog.prototype.eat = function() {
    return `${this.name} the ${this.breed} is eating dog food`;
};

const dog = new Dog('Buddy', 'Golden Retriever');
console.log(dog.eat());   // "Buddy the Golden Retriever is eating dog food"
console.log(dog.sleep()); // "Buddy is sleeping" (inherited)
console.log(dog.bark());  // "Buddy says woof!" (dog-specific)
```

### ES6 Classes (Syntactic Sugar):

```javascript
class Animal {
    constructor(name) {
        this.name = name;
    }

    eat() {
        return `${this.name} is eating`;
    }

    sleep() {
        return `${this.name} is sleeping`;
    }
}

class Dog extends Animal {
    constructor(name, breed) {
        super(name); // Call parent constructor
        this.breed = breed;
    }

    bark() {
        return `${this.name} says woof!`;
    }

    eat() {
        return `${this.name} the ${this.breed} is eating dog food`;
    }
}

const dog = new Dog('Buddy', 'Golden Retriever');
console.log(dog.eat());   // "Buddy the Golden Retriever is eating dog food"
console.log(dog.sleep()); // "Buddy is sleeping"
```

### Prototype Methods and Properties:

```javascript
function User(name) {
    this.name = name;
}

// Instance method (on each instance)
User.prototype.getName = function() {
    return this.name;
};

// Static method (on constructor)
User.createGuest = function() {
    return new User('Guest');
};

// Static property
User.version = '1.0';

const user = new User('Alice');
console.log(user.getName());        // "Alice"
console.log(User.createGuest().getName()); // "Guest"
console.log(User.version);          // "1.0"
```

### Checking Prototype Chain:

```javascript
console.log(user.__proto__ === User.prototype);        // true
console.log(User.prototype.__proto__ === Object.prototype); // true
console.log(Object.prototype.__proto__);               // null

// Modern way to check prototype
console.log(Object.getPrototypeOf(user) === User.prototype); // true

// Check if property exists on prototype chain
console.log(user.hasOwnProperty('name'));              // true (own property)
console.log(user.hasOwnProperty('getName'));           // false (inherited)
console.log('getName' in user);                        // true (in prototype chain)
```

## Real React Examples:

### 1. **Component Inheritance with Prototypes:**
```javascript
function BaseComponent(props) {
    this.props = props;
    this.state = {};
}

BaseComponent.prototype.setState = function(newState) {
    this.state = { ...this.state, ...newState };
    this.forceUpdate && this.forceUpdate();
};

BaseComponent.prototype.forceUpdate = function() {
    // Trigger re-render logic
    console.log('Component re-rendering');
};

function ButtonComponent(props) {
    BaseComponent.call(this, props);
    this.state = { clicked: false };
}

ButtonComponent.prototype = Object.create(BaseComponent.prototype);
ButtonComponent.prototype.constructor = ButtonComponent;

ButtonComponent.prototype.handleClick = function() {
    this.setState({ clicked: true }); // Inherited method
};
```

### 2. **Mixin Pattern with call/apply:**
```javascript
const eventMixin = {
    on(eventName, handler) {
        if (!this._eventHandlers) this._eventHandlers = {};
        if (!this._eventHandlers[eventName]) {
            this._eventHandlers[eventName] = [];
        }
        this._eventHandlers[eventName].push(handler);
    },

    off(eventName, handler) {
        const handlers = this._eventHandlers?.[eventName];
        if (!handlers) return;
        const index = handlers.indexOf(handler);
        if (index > -1) handlers.splice(index, 1);
    },

    trigger(eventName, ...args) {
        if (!this._eventHandlers?.[eventName]) return;
        this._eventHandlers[eventName].forEach(handler => {
            handler.apply(this, args);
        });
    }
};

// Apply mixin to any object
function Menu() {
    // Apply mixin methods to this object
    Object.assign(this, eventMixin);
}

const menu = new Menu();
menu.on('select', function(item) {
    console.log('Selected:', item);
});

menu.trigger('select', 'File'); // "Selected: File"
```

### 3. **React Class Component with bind() and Prototypes:**
```javascript
class TodoApp extends React.Component {
    constructor(props) {
        super(props);
        this.state = { todos: [] };

        // Bind methods to preserve 'this'
        this.addTodo = this.addTodo.bind(this);
        this.removeTodo = this.removeTodo.bind(this);
    }

    addTodo(text) {
        this.setState(prevState => ({
            todos: [...prevState.todos, { id: Date.now(), text }]
        }));
    }

    removeTodo(id) {
        this.setState(prevState => ({
            todos: prevState.todos.filter(todo => todo.id !== id)
        }));
    }

    render() {
        return (
            <div>
                <TodoForm onAdd={this.addTodo} />
                <TodoList
                    todos={this.state.todos}
                    onRemove={this.removeTodo}
                />
            </div>
        );
    }
}

// TodoForm component
class TodoForm extends React.Component {
    constructor(props) {
        super(props);
        this.state = { text: '' };
        this.handleSubmit = this.handleSubmit.bind(this);
    }

    handleSubmit(e) {
        e.preventDefault();
        this.props.onAdd(this.state.text);
        this.setState({ text: '' });
    }

    render() {
        return (
            <form onSubmit={this.handleSubmit}>
                <input
                    value={this.state.text}
                    onChange={(e) => this.setState({ text: e.target.value })}
                />
                <button type="submit">Add Todo</button>
            </form>
        );
    }
}
```

## Key Concepts:

### call/apply/bind:
- **`call()`**: Execute immediately with individual arguments
- **`apply()`**: Execute immediately with array of arguments  
- **`bind()`**: Create bound function for later execution

### Prototypes:
- **Prototype Chain**: Objects inherit from their prototype
- **Constructor Functions**: Create objects with shared methods
- **Inheritance**: Child objects can inherit from parent prototypes
- **Performance**: Prototype methods are shared, saving memory

### Interview Tip:
*"call/apply/bind control function context, while prototypes enable inheritance. call() and apply() execute immediately, bind() creates a new function. Prototypes allow method sharing and create the inheritance chain JavaScript uses for property lookup."*