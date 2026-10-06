Use a class when something has data and behaviour. For example:

|Class|Data|Behaviour|
|---|---|---|
|BankAccount|balance|deposit()|
|User|name|greet()|
|ApiClient|baseUrl|get()|
|Timer|seconds|start()|

Constructor and methods

```js
/*
1. constructor runs once when you do "new"
2. this.balance stores the data on the instance
3. methods go straight in the class body, no "function" keyword
*/

class BankAccount {
    constructor(owner, balance = 0) {
        this.owner = owner;
        this.balance = balance;
    }

    deposit(amount) {
        this.balance += amount;
    }

    showBalance() {
        console.log(`${this.owner}: ${this.balance}`);
    }
}

const account = new BankAccount("Bob");

account.deposit(100);
account.showBalance();  // output: Bob: 100
```

this refers to the instance, and it gets lost if you detach the method

```js
class Counter {
    constructor() {
        this.count = 0;
    }

    increment() {
        this.count++;
        console.log(this.count);
    }
}

const c = new Counter();

c.increment();  // output: 1

const detached = c.increment;
// detached();  // TypeError: Cannot read properties of undefined

// fix it by wrapping in an arrow so "this" stays bound
const safe = () => c.increment();
safe();  // output: 2
```

extends and super

```js
class SavingsAccount extends BankAccount {
    constructor(owner, balance, rate) {
        super(owner, balance);   // must call super before using this
        this.rate = rate;
    }

    addInterest() {
        this.balance += this.balance * this.rate;
    }

    showBalance() {
        super.showBalance();     // call the parent version too
        console.log(`rate: ${this.rate}`);
    }
}

const savings = new SavingsAccount("Alice", 1000, 0.05);

savings.addInterest();
savings.showBalance();
// output: Alice: 1050
// output: rate: 0.05
```

Private fields with #

```js
class BankAccount {
    #pin = 1234;   // can't be touched from outside

    check(guess) {
        return guess === this.#pin;
    }
}

const account = new BankAccount();

console.log(account.check(1234));  // output: true
// console.log(account.#pin);      // SyntaxError
```
