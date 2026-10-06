Use classes when something has data and behaviour. For example:

|Class|Data|Behaviour|
|---|---|---|
|BankAccount|balance|deposit()|
|ApiClient|baseUrl|get()|
|AuthStore|token|login()|
|Cart|items|addItem()|

Class with typed fields

```ts
class BankAccount {
  balance: number;
  owner: string;

  constructor(owner: string, balance: number) {
    this.owner = owner;
    this.balance = balance;
  }

  deposit(amount: number): void {
    this.balance += amount;
  }

  showBalance(): number {
    return this.balance;
  }
}

const account = new BankAccount("Bob", 100);
account.deposit(50);
account.showBalance(); // 150
```

public, private and readonly

```ts
class BankAccount {
  public owner: string;
  private balance: number;      // only usable inside the class
  readonly createdAt: string;   // set once in the constructor, never again

  constructor(owner: string) {
    this.owner = owner;
    this.balance = 0;
    this.createdAt = "today";
  }
}

const account = new BankAccount("Bob");

account.balance;              // Error: Property 'balance' is private and only accessible within class 'BankAccount'
account.createdAt = "later";  // Error: Cannot assign to 'createdAt' because it is a read-only property
```

Constructor shorthand, the modifier declares and assigns the field for you

```ts
class BankAccount {
  // same as declaring the fields then doing this.owner = owner
  constructor(
    public owner: string,
    private balance: number = 0,
  ) {}

  deposit(amount: number): void {
    this.balance += amount;
  }
}

const account = new BankAccount("Bob");
account.owner; // "Bob"
```

Implementing an interface

```ts
interface Storage {
  save(key: string, value: string): void;
  load(key: string): string | null;
}

// implements is checked at compile time, a bit like impl Trait for Struct
class LocalStorage implements Storage {
  save(key: string, value: string): void {
    localStorage.setItem(key, value);
  }

  load(key: string): string | null {
    return localStorage.getItem(key);
  }
}

// leaving out load() gives:
// Error: Class 'LocalStorage' incorrectly implements interface 'Storage'
```
