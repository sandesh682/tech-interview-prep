
# 🧩 Q1: What is Binary Tree?

A Binary Tree is a hierarchical data structure where each node has at most 2 children.

---

## 🌲 Types of Binary Trees (with Examples)

### 1. Full Binary Tree

👉 Every node has **0 or 2 children**

#### ✅ Full Tree

```
      1
     / \
    2   3
   / \
  4   5
```

#### ❌ Not Full

```
      1
     / \
    2   3
   /
  4
```

(Node 2 has only one child)

---

### 2. Complete Binary Tree

👉 All levels filled except last, last filled **left to right**

#### ✅ Complete

```
      1
     / \
    2   3
   / \  /
  4  5 6
```

#### ❌ Not Complete

```
      1
     / \
    2   3
     \     ← gap on left
      5
```

---

### 3. Perfect Binary Tree

👉 All levels completely filled

#### ✅ Perfect

```
      1
     / \
    2   3
   / \ / \
  4  5 6  7
```

#### ❌ Not Perfect

```
      1
     / \
    2   3
   / \
  4   5
```

---

### 4. Balanced Binary Tree

👉 Height difference ≤ 1 for every node

#### ✅ Balanced

```
      1
     / \
    2   3
   /
  4
```

#### ❌ Not Balanced

```
      1
     /
    2
   /
  3
```

---

### 5. Skewed Tree

#### Left Skewed

```
    1
   /
  2
 /
3
```

#### Right Skewed

```
1
 \
  2
   \
    3
```

---

## ❓ Why BST is NOT in "Types"?

👉 Important conceptual clarity:

- **Binary Tree types** = based on _structure_
    
- **BST (Binary Search Tree)** = based on _ordering rule_
    

### BST Rule:

```
Left < Root < Right
```

👉 So:

- A BST can be **full / complete / balanced**
    
- OR it can be **skewed (bad BST)**
    

➡️ That’s why BST is a **different category**, not a "type"

---

## 🔥 Priority Queue (PQ) Connection

👉 Most interview confusion comes here.

### Key Idea:

- A **Priority Queue** is often implemented using a **Heap**
    
- A Heap is a **Complete Binary Tree**
    

---

### 🧱 Min Heap Example

```
      1
     / \
    3   6
   / \
  5   9
```

Properties:

- Complete tree
    
- Parent ≤ children
    

---

### 🧱 Max Heap Example

```
      9
     / \
    5   6
   / \
  1   3
```

Properties:

- Complete tree
    
- Parent ≥ children
    

---

### ⚡ Why Complete Tree?

Because we store heap in array:

```
Index:
i → node
2i+1 → left
2i+2 → right
```

👉 Works only when tree is **complete**

---

## 🧠 Key Takeaways

- Full ≠ Complete ≠ Perfect (VERY common confusion)
    
- Complete tree is most practical (used in heaps)
    
- BST is about **ordering**, not shape
    
- Heap = Complete Binary Tree + Heap Property
    

---
