---
title:  Leetcode - Reverse Polish with STL
mathjax: true
layout: post
date:   2024-01-29 00:00:30 +0300
description: Solving leetcode with the STL
tags:   [C++]
image:  '/media/posts_thumb/cppleetcode.png'
---



# Leetcode Reverse Polish Notation

So this is a pretty cool medium [leetcode](https://leetcode.com/problems/evaluate-reverse-polish-notation/description) problem. It goes like this:

> Given an array of tokens that represent an arithmetic expression in Reverse Polish Notation, evaluate the expression. The result **MUST** be an integer


Alrighty so, there are a few ways to tackle this problem, the traditional way is to use a stack. Here's how it goes:


- Iterate through the tokens
- if the current token is a number push it into the stack
- otherwise the token is an operation **(+, - , /, \*)** so pop two values from the stack and perform the operation indicated by the current token
- push the result back into the stack
- At the end, the stack should contain only one element which is the final result


## Example

Let's say our input consists of the following tokens: 

> 2 1 + 3 \*

First things first, we create our stack:


![image 1](/media/Leetcode-Reverse-Polish-with-STL/stack1.png) 

We keep pushing the tokens until we find something that's not a number:

![image 2](/media/Leetcode-Reverse-Polish-with-STL/stack2.png) 

Now we need to pop two elements from the stack to perform our arithmetic. The top of the stack is **always** going to be the right hand side of the equation. After popping and doing the calculation, push the result back:


![image 3](/media/Leetcode-Reverse-Polish-with-STL/stack3.png) 


Continuing, we find another number so we push it as well

![image 4](/media/Leetcode-Reverse-Polish-with-STL/stack4.png) 

Finally, our last token is an operation, popping two elements from the stack gives us 9:

![image 5](/media/Leetcode-Reverse-Polish-with-STL/stack5.png) 

All that's left now is to push this value into the stack and that's our final result:

![image 6](/media/Leetcode-Reverse-Polish-with-STL/stack6.png) 


## **C++ solution**

As I said the traditional way to solve this is simply using a stack and some if statements to check which operation should be performed. But we can sprinkle some STL goodness here.


![sprinkleem](/media/sprinkle.gif)

This solution uses a hashmap and a lambda to perform the lookup on the arithmetic operation and to pop the values from the stack.

The hashmap key is the desired operation and its value is the result of that operation applied to the two values from the top of stack.

Just one more thing, we need to take care of inputs with **negative numbers** which is shown below.



```c++

int evalRPN( std::vector<std::string>& tokens )
{
    std::stack<int> stack;
    std::unordered_map<char, std::function<int( int, int )>> map =
    {
        {'+', []( int lhs, int rhs ) { return lhs + rhs; }},
        {'-', []( int lhs, int rhs ) { return lhs - rhs; }},
        {'*', []( int lhs, int rhs ) { return lhs * rhs; }},
        {'/', []( int lhs, int rhs ) { return lhs / rhs; }}
    };

    auto pop = [ &stack, &map ]( char op )
        {
            if( stack.size() >= 2 ) [[likely]]
            {
                auto rhs = stack.top();
                stack.pop();
                auto lhs = stack.top();
                stack.pop();

                return map[op]( lhs, rhs );
            }

        };

    for( const auto c : tokens )
    {
        // take care of negative numbers
        if( std::isdigit(c[0]) || c.length() > 1 && std::isdigit(c[1]))
        {
            bool isNegative = c[0] == '-';

            auto numString = isNegative ? c.substr( 1 ) : c;
            auto num = std::stoi( numString );

            stack.push( isNegative ? -num : num );
        }
        else
        {
            auto result = pop( c[0] );
            stack.push( result );
        }
    }

    if( !stack.empty() ) [[likely]]
    {
        auto result = stack.top();
        return result;
    }

    return 0;
}
```

## **Any room for improvement?**


Well, sure. There's one thing I've been seeing people do to get the lowest latency possible on leetcode. It boils down to :


- Creating a global lambda and calling it immediately 
- Desync C and C++ streams
- Untie the cin and cout streams
- Parse the input string manually

```c++

int init = [] {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);
     ofstream out("user.out");
     for (string s; getline(cin, s); out << '\n') 
     {
	      // parse input
      }

        // continue with the task here...
     
     }(); // call it immediately
```

The call to **ios_base::sync_with_stdio(false);** disables the synchronization between the C and C++ standard streams. This allows for independent buffers and a superior performance, although not by much, at least not on leetcode. I guess it depends on what kind of input is given, sometimes this does wonders, sometimes not so much.

And **cin.tie(nullptr);** unties cin from cout. Both **cin** and **cout** are usually tied, meaning that one operation on one of them will flush the other. This is done to avoid unnecessary flushing. So after this, reading from **cin** will no longer automatically flush **cout** .  After this, we'd need to flush them manually, but  there's still a performance boost to be gained by doing it.

Also it's possible to change our **std::unordered_map**  to a **std::unordered_set**




