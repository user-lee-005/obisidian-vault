There are 4 levels of characters
1. [[#ASCII]]
2. [[#BMP]] (Basic Multilingual Plane)
3. [[#Supplementary Characters]]
4. [[#Grapheme Clusters]]
---
## ASCII
- Example: A B C 1 2 3
- "A".lenght() == 1
---
## BMP
- Examples:
```java
é
ü
ñ
₹
தமிழ்
中
```
- These fit into single character in Java
- "é".length() == 1
---
## Supplementary Characters
- Examples:
```java
🚀
🦊
😀
🎉
```
- These dont fit into a single character in java
- They are stored as 
```
High Surrogate
Low Surrogate
```
- "🚀".length() == 2
- To solve the issue in LANG-1770, i need to understand the below:
```java
Character.isHighSurrogate()
Character.isLowSurrogate()

String.codePointCount()

String.offsetByCodePoints()
```
---
## Grapheme Clusters
- Examples:
```java
👨‍👩‍👧‍👦
👍🏽
👩🏻‍💻
```
- These are built from multiple code points, for example
```java
👨‍👩‍👧‍👦 = 👨 + ZWJ + 👩 + ZWJ + 👧 + ZWJ + 👦
```
- This is where the things get complicated
- For LANG-1770, understand them, do not support fully, add test case for showing their behaviour
---
## Code point

- A **code point** (one logical character) is a Unicode character represented by a unique number.
```
|Character|Unicode Code Point|
|---------|------------------|
|A        |U+0041            |
|₹        |U+20B9            |
|🚀       |U+1F680           |
|😀       |U+1F600           |
```

- Java String stores text as UTF-16
- UTF-16 uses: 1. 1 char (16 bits) more common for chars, 2. 2 chars (a surrogate pair) for characters outside basic multilingual plane (such as emojis)
## Code Unit
- Each unit in the char array of a string
```java
🚀 -> U+1F680 // 1 code point, 2 code units (one low surrogate and one high surrogate) represented in the array as [high surrogate][low surrogate]
```
---
### Main methods to look into
- length:
```java
rocket.length(); // returns utf-16 code units
```
- codePointCount:
```java
rocket.codePointCount(0, rocket.length()); // gets in the starting and ending index as the parameters and gives the number of code points present as a result
```
- offsetByCodePoints:
	- A **Code Point** represents one actual, complete logical character, regardless of whether it takes one or two `char`s to store it.
	- If you just added a number to an `index` (e.g., `index + 3`), you might accidentally land right in the middle of a surrogate pair (cutting an emoji in half). The `offsetByCodePoints` method prevents this by safely jumping over whole characters.
```java
public int offsetByCodePoints(int index, int codePointOffset) { 
	return Character.offsetByCodePoints(this, index, codePointOffset); 
}
String s = "🚀🚀🚀";  
int idxAfterTheOffset = s.offsetByCodePoints(0, 1); // result 2

- **The Main Description:**
    
    - _"Returns the index... offset from the given `index` by `codePointOffset` code points."_ -> This calculates where you will end up if you start at `index` and move forward (or backward) by a specific number of actual characters.
        
    - _"Unpaired surrogates... count as one code point each."_ -> If the method encounters broken or invalid Unicode (a half-character missing its partner), it won't crash; it will just count that broken piece as one single step.
        
- **`@param index`**: The position in the string where you want to start counting from.
    
- **`@param codePointOffset`**: How many logical characters you want to move. A positive number moves you forward toward the end of the string, and a negative number moves you backward toward the beginning.
    
- **`@return`**: The new, absolute index within the string. (Note: If you step over a lot of emojis, an offset of `2` might actually increase your absolute index by `4`).
    
- **`@throws IndexOutOfBoundsException`**: The method will throw an error if you try to do the impossible:
    
    - If your starting `index` is less than 0 or greater than the string's length.
        
    - If you tell it to step forward, but there aren't enough characters left in the string before it hits the end.
        
    - If you tell it to step backward (negative offset), but there aren't enough characters before you hit the very beginning of the string.
        
- **`@since 1.5`**: This method was added in Java 5 (released in 2004), which is when Java overhauled its system to properly support supplementary Unicode characters like emojis.
```
---

> [!info] Findings based on the Java String class
> Java String indexing is based on UTF-16 code units rather than Unicode code points. As a result, operations that use raw character indexes (such as substring) can split surrogate pairs and produce invalid Unicode strings. Code-point-aware APIs such as offsetByCodePoints avoid this issue.

---
