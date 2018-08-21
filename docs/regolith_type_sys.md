# Regolith 
## Adding a static type system to Lua (Part 1)

When people ask my what my favorite programming language is I'm often quick to say C#, and can talk the ears off of anyone who will listen about all its great aspects. My *second favorite* language though is one I get to talk about way less frequently: Lua. 

People have a lot of different reactions when I list Lua as one of my favorite languages - most common of which are "you can't be serious" and "what is Lua?" For the latter my response is often "you know the book [Javascript: The Good Parts](http://shop.oreilly.com/product/9780596517748.do)? If you take that book and make a language out of it you get Lua."

At first glance the differing syntax of the two languages might convince someone otherwise. Javascript has a C inspired syntax, while Lua is more english-like. Beyond that though both languages are dynamic, interpreted scripting languages with a hashmap type as the core of its "prototype" object orientation.

> A very contrived function with a local dynamic value in Lua
```lua
function getValue()
    local condition = true
    if condition then
        condition = "Hello World"
    else 
        condition = 6
    end
    return condition
end
```

> An identical function in Javascript
```js
function getValue() {
    let condition = true;
    if (condition) {
        condition = "Hello World";
    } else {
        condition = 6;
    }
    return condition;
}
```

Since their creation JavaScript's near monopoly on browser interactivity has gifted it a massive userbase. With the support of this wide adoption JavaScript has left Lua in the dust in terms of language features and runtime support. Meanwhile, Lua has maintained its little niche as an easy to learn scripting language embeddable into C/C++ programs (exactly the niche JavaScript was created to fill 2 years later!) and is most commonly found in the games industry as a tool for modders (example: [factorio](https://lua-api.factorio.com/latest/)). For JavaScript, adoption by literally millions of developers has enabled things like excellent package management, async patterns support, and transpilation and linting tools.

When Lua was first developed in 1993 the predominant application development language was C - and Lua was designed to be easily integratable into C applications as an embedded language. Now in 2018 the role of C has shifted away from general purpose application development in favor of languages like Java, C#, and Python. Desipite that, if I want to embed Lua in a dotnet application I need to reference a C DLL and make extern calls. For years I've entertained the idea of reimplementing Lua in C# in a way that would allow it to "natively" understand dotnet objects and patterns (like Enumerators!), allow Lua to adopt Nuget as its package manager, and give Lua access to the dotnet runtimes support and optimizations. 

Additionally, implementing Lua in a high level managed language like C# would make it easier, faster, and safer to add advanced tooling features like linting, transpilation - and advanded language features like a type system. When thinking about a type system for Lua the natural inspiration would be TypeScript, although the addition of types to Lua also provides a unique opportunity to cherry pick some cool type concepts from other languages as well.

## Here is what types in Lua might look like

Lua already has "objects" the same way JavaScript does, in the form of tables. In both languages creating an object is creating a hash map, and constructors are functions that return maps with some common set of features. In JavaScript those common features are managed with each maps reference to a "parent" map called a `prototype`, where in Lua the same behavior is done using `metatables` to override the default table key resolution behavior and point missing values to another table.

> A simple 'object' constructor in lua with data and behavior using a clojure to simulate a private field and without using metatables to mimick inheritance 
```lua
function GetPerson(name)
    local pay = math.random(50, 100)
    return {
        Name = name,
        Coins = 0,
        Pay = function (self)
            self.Coins = self.Coins + pay
        end,
        CanBuyABoat = function (self)
            return self.Coins > 1000000
        end
    }
end
local developer = GetPerson("Kelson")
developer:Pay()
print(developer:CanBuyABoat())
```

> A similar 'object' definition in JavaScript
```js
function GetPerson(name) {
    let pay = Math.random(500, 1000);
    return {
        Name: name,
        Coins: 0,
        Pay: function () {
            this.Coins += pay;
        },
        CanBuyABoat: function () {
            return this.Coins > 1000000;
        }
    };
}
let developer = GetPerson("Kelson");
developer.Pay();
console.log(developer.CanBuyABout());
```

In TypeScript the same object might look something like this,
```typescript
class Person {
    _pay: number;
    Name: string;
    Coins: number;
    constructor(name: string) {
        this._pay = Math.random(600, 1100);
        this.Coins = 0;
        this.Name = name;
    }
    Pay() {
        this.Coins += this._pay;
    }
    CanBuyABoat() : boolean {
        return this.Coins > 1000000;
    }
}

let developer = new Person("Kelson");
developer.Pay();
console.log(developer.CanBuyABoat());
```

What I'm exploring implementing in Regolith however looks a little less like traditional classes and a little more like composible interfaces.

```lua
type Person has
    string Name
    number Coins
    (self) -> () Pay
    (self) -> bool CanBuyABoat
end

local function GetPerson(string name) -> Person
    local pay = 0
    return {
        Name = name,
        Coins = 0,
        Pay = function(self)
            self.Coins = self.Coins + 100
        end,
        CanBuyABoat = function(self) -> bool
            return self.Coins > 1000000
        end
    }
end

local developer = GetPerson("Kelson")
developer:Pay();
print(developer:CanBuyABoat())
```

In this case a compile-time tool can parse the GetPerson function and see that it is defined as returning an object maching the type Person. It can enforce this rule by traversing the syntax tree of the function to make sure every returned value is a table that has a field `Name` assigned to some string, a field `Coins` assigned to some number, a field `Pay` assigned to a function that takes a Person, and a field `CanBuyABoat` assigned to a function that maps a Person to a boolean value. 

This raises some questions about type enforecement and objects becoming invalid, so let me explore a little further to show how those cases might be handled. 

An important aspect of a type system is how that system allows types to be composed and structured. Here are some examples of how Regolith types might be composed using type expressions.

> Inheritence and composition

```lua
type Moveable has 
    string Location
    (self, string) -> () Move    
end

type Wearable has
    string Description
end

type Person is Moveable has
    string Name
    Wearable Outfit    
    (self, Wearable) -> () ChangeOutfit
end
```

The best way I've heard the OOP concepts of inheritance and composition described is that "composition describes what an object has and inheritence describes what an object is," so those seem like natural syntatic elements out of which to build a type system for the beginner friendly, english friendly language Lua. In a traditional OO language like C# a class definintion would also describe what a type *does,* but keeping in the Lua/JavaScript pattern of objects being fancy hash maps, behavior is something an object *has.* When you dive further down the idea of types being sets of "has" relations inheritance becomes equivilent to the *union* set operation. 

```lua
type Chargable has
    (self, number) -> bool charge
end

type Virtual has
    string apiToken
    string apiUrl
end

-- type union with 'and'
type Card is Chargable and Virtual has    
    string currencyCode    
end

type Cash is Chargable has
    number ammount
    string currencyCode    
end

type BitCoin is Chargable and Virtual has
    number ammount
    datetime time
end

-- type intersection with 'or'
type AcceptedPayments is Card or Cash end
```

In Part 2 I'll dive more into how these type rules might be organized and enforced, and how they would fascilitate interopability with dotnet libraries. 