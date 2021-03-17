<h1>Callable Constructors&trade</h1><!-- omit in toc -->
 
| &nbsp;           | &nbsp;                                                   |
| ---------------- | -------------------------------------------------------- |
| Document Number: | **PXXXX**                                                |
| Date             | 2021-APR-??                                              |
| Audience         | WG21                                                     |
| Author           | Dusan B. Jovanovic ( [dbj@dbj.org](mailto:dbj@dbj.org) ) |


- [1. Abstract](#1-abstract)
- [2. Motivation](#2-motivation)
- [3. Synopsis](#3-synopsis)
  - [3.1. Callable Constructors and Unfinished instances](#31-callable-constructors-and-unfinished-instances)
- [4. The language extension](#4-the-language-extension)
  - [4.1. Usage](#41-usage)
  - [4.2. Legacy](#42-legacy)
- [5. Conclusion](#5-conclusion)


"There are two ways of constructing a software design: One way is to make it so simple that there are obviously no deficiencies, and the other way is to make it so complicated that there are no obvious deficiencies. The first method is far more difficult." -- [C.A.R. Hoare](https://en.wikiquote.org/wiki/C._A._R._Hoare)


## 1. Abstract

This paper describes very standard  C++ "callable constructors", by proposing a core language change that is fundamental but in the same time non breaking.

## 2. Motivation

<!-- In the constructor the runtime uses type data to determine how much space is needed to store an object instance in memory. After this space is allocated, the constructor is called as an internal part of the instantiation and initialization process to initialize the contents of the instance.

Then the constructor exits, the runtime returns the newly-created instance. So the reason the constructor doesn't return a value is because it's not called directly, it's called by the memory allocation and object initialization code. -->

On a bit higher level, modern OO languages (GO, RUST, etc) have decided to avoid constructors, with good reasoning. Also, the factory functions vs constructors as a general idea seemed very plausible in many ways. In e.g. C, one can (and does) a lot of OO without constructors.

Standard C++ constructors only way to signal the outcome, is to throw an exception. Projects where exception are mandated to be non existent are using factory methods to create class instances. That has effectively created another C++ dialect and fragmented the community.

Last but not the least: Lambda is kind-of-a callable constructor. For an "invisible" class.

## 3. Synopsis

**Vocabulary**
| Term                 | Meaning                                         |
| -------------------- | ----------------------------------------------- |
| C Ctor               |
| callable constructor | Callable Constructor                            |
| CC Class             | Class having one or more Callable Constructors  |
| CC Struct            | Struct having one or more Callable Constructors |

CC Class can have a mixture of callable and non-callable (aka "normal") constructors. 

**Specimen CC Class:**

```cpp
using std::string;
using std::string_view;
// CC Struct
struct person 
{
string data_ ; 

// destructors have always been callable
~person () { if (! data_.empty() ) data_ = ""; }
// thee is no return statement 
// this constructor is not callable
person () noexcept : data_("") { }

// callable constructor
// constructor signature is same as ever
person ( string_view new_name_) : data_ (new_name_) 
{  
  // callable constructor only difference 
  // vs standard constructor is it contains 
  // one or more return statements
  // type returned is deduced exactly as for functions
  // where the return type is declared as auto
 return data_;
}

// if callable constructor is not called
// outcome is exactly the same as for standard constructor
// the instance of the class constructed
person ( string_view name, string_view surname ) noexcept : data_(name) 
{
  if ( surname.size() > 0 ) {
    data_.append("|"); data.append(surname);
    return true ;
  }
    return false ;
}

person ( bool flag ) noexcept : data_("") 
{
  // return statement makes this constructor callable
    return ! flag ;
}

// this will be returned from a callable constructor
// as a pointer
person ( float, float, float ) noexcept : data_("") 
{
  // extreme caution advised
    return this ;  
}

// *this returned from a callable constructor
// as a reference effectively delivers a
// factory method constructor
person ( int id, string_view name ) noexcept : data_("") 
{
  // returned as reference
  // caution advised
    return *this ; 
}

// copy constructor can not be callable
person ( person const & ) noexcept = default ;
person & operator = ( person const & ) noexcept = default ;

// move constructor can not be callable
person ( person && another_ ) noexcept = default ;
person & operator = ( person && other) noexcept = default ;

} ; // eof person
```

Constructor return type rules are the same as return rules anywhere else in the standard C++.

### 3.1. Callable Constructors and Unfinished instances

In case of returning a valstat prematurely i.e.from an unfinished object ctor, compiler should be able to catch that as an error.

## 4. The language extension

Keyword 'auto` will be paired with the constructible type to mark the constructor calling, for both humans and compilers. This is to differentiate it from the standard constructor syntax, visually for humans and deterministically for compilers.

Constructor Call formal definition:
```
<constructible type> auto <standard constructor syntax> ;
```
Constructor Call outcome is the same as function call outcome.
```
auto value = <constructible type> auto <standard constructor syntax> ;
```
- Implicit default constructors are not callable
- Constructor with no returns are not callable
- Deduction of the type returned is the same as for functions returning auto

Basically constructor call has the same effect as function call.

### 4.1. Usage 

```cpp
// standard construction
// obj is of a type person
person obj{ };  

// standard construction
// this ctor has return statements 
// it is used in a standard way
// thus peter is of a type person
person peter ( "Peter", "Pan" );  

// constructor call
// 'auto' is right of the type name
// this constructor is implemented
// to return std::string, if called
std::string full_name = person auto ( "Misteria", "" );  

// default ctor has no return statements
// it is not callable
// this does not compile
// auto whatever = person auto ()  ; 

// this does not compile too
// using type_returned_ = decltype(person auto() ) ;

// callable ctor return type
using ctor_type_returned_ = 
decltype( person auto( std::string_view, std::string_view )) ;

// type of the value returned
// is the type of the ctor call 
auto function ( bool flag ) 
noexcept -> decltype( person{bool} ) 
{
    return person auto (flag); // constructor call
}
```

Callable constructors returning *this are effectively factory methods.

```cpp
// when called this constructor makes a person and returns 
// reference to it
person const & jim = person auto ( 42, "Jim" );

// copy ctor can not be callable 
person stable_jim( jim ) ;
```
Of course we can design and use more elaborate types to have the full information from inside the callable constructor.

### 4.2. Legacy

One is free to construct T on heap and do traditional constructions as ever before. The legacy code is unaffected.

```cpp
// same as ever before
// can not call constructors when new-ing the instances
person * persons = new person[3]{"Jim", "Beam", "Slim"} ;

// container of 3 persons
std::vector<person> persons_v(3) ;
```

`new` can no be mixed with a new usage of `auto` in order to call constructors.

## 5. Conclusion

Can this mechanism be abused? Anything in C++ can be abused. Standard constructor paradigm can be abused. Callable constructors can be abused too.

This language extension would not break any existing code. std lib as we know it today will "just work".

*EOF*