# Python Fundamentals Learning Notes

**Learner:** Onyekachi Nkuma  
**Date:** 15 November 2026  
**Environment:** Ubuntu 26.04.1 LTS (WSL)  
**Python Version:** Python 3.14.4

## 1. Introduction

Python is a programming language used for automation, scripting,
application development, data processing and cloud engineering.

In Cloud/DevOps engineering, Python is useful for:
- Automating repetitive tasks.
- Working with AWS services using Boto3.
- Processing JSON data.
- Interacting with REST APIs.
- Managing cloud infrastructure.

## 2. Python Variables and Data Types

A variable stores information that a Python program can use.

Example:

```python
name = "Onyekachi"
age = 25
height = 1.75
is_learning = True
```

To display the variables:

```python
print(name)
print(age)
print(height)
print(is_learning)
```
## 3. Arithmetic Operators
Python supports arithmetic operations:
```python
print(10 + 3)  # Addition: 13
print(10 - 3)  # Subtraction: 7
print(10 * 3)  # Multiplication: 30
print(10 / 3)  # Division
print(10 // 3) # Floor division: 3
print(10 % 3)  # Modulus: 1
print(10 ** 3) # Exponentiation: 1000
```
## 4. Comparison Operators
Comparison operators compare values and return `True` or `False`.
```python
print(10 > 5)
print(10 < 5)
print(10 == 10)
print(10 != 5)
print(10 >= 10)
print(5 <= 10)
```
## 5. Boolean Values
Boolean values are `True` and `False`. They are useful when a program needs to make decisions.
## 6. Conditional Logic: if
An `if` statement runs a block of code when its condition is true.
```python
age = 20
if age >= 18:
    print("You are an adult")
```
## 7. Conditional Logic: else
`else` runs when the `if` condition is false.
```python
age = 15
if age >= 18:
    print("You are an adult")
else:
    print("You are not an adult")
```
## 8. Conditional Logic: elif
`elif` checks another condition when the earlier `if` condition is false.
```python
score = 65
if score >= 80:
    print("Excellent")
elif score >= 60:
    print("Good")
else:
    print("Keep practising")
```
## 9. Indentation and Code Blocks
Python uses indentation (spaces at the beginning of a line) to group statements into code blocks.
```python
score = 80
if score >= 50:
    print("You passed")
    print("Well done")
print("Program finished")
```
The two indented statements belong to the `if` block. The final print statement is outside the block, so it runs regardless of the condition.
## 10. Logical Operators
- `and`: all conditions must be true.
- `or`: at least one condition must be true.
- `not`: reverses a Boolean value.
Examples:
```python
age = 25
has_permission = True
if age >= 18 and has_permission:
    print("Access granted")
```
```python
is_admin = False
has_emergency_code = True
if is_admin or has_emergency_code:
    print("Access granted")
```
```python
is_blocked = False
if not is_blocked:
    print("Account is active")
```
## 11. Combining Conditions
We can combine comparison and logical operators to make decisions based on multiple conditions.
```python
age = 25
has_permission = True
is_blocked = False
if age >= 18 and has_permission and not is_blocked:
    print("Access granted")
else:
    print("Access denied")
```
If the account is blocked, access is denied even if the other conditions are true.
## 12. Running Python in Ubuntu (WSL)
We can run Python code directly from the Ubuntu terminal using a heredoc:
```bash
python3 - <<'PY'
print("Hello, Python")
PY
```
The `python3` command starts Python. The `PY` markers tell Bash where the Python input begins and ends.
## 13. Cloud and DevOps Applications
These fundamentals will help with future tasks such as:
- Writing Python automation scripts.
- Reading and processing JSON responses.
- Calling REST APIs.
- Using Boto3 to interact with AWS.
- Checking cloud resources and automating routine operations.
## 14. Learning Progress
| Topic | Status |
|---|---|
| Variables and data types | Completed |
| Arithmetic operators | Completed |
| Comparison operators | Completed |
| Boolean values | Completed |
| if, else and elif | Completed |
| Indentation and code blocks | Completed |
| Logical operators | Completed |
| Combining conditions | Completed |
## 15. Next Learning Topics
1. Python scripts and running `.py` files.
2. JSON fundamentals and working with JSON in Python.
3. Python functions and error handling.
4. REST APIs and HTTP requests.
5. AWS automation using Boto3.
---
**Checkpoint:** Python conditional logic completed on 15 November 2026.
**AWS safety:** The Python exercises in this section were local learning exercises. No AWS resources were created, modified or deleted.
