### **Quick YAML Tutorial (with Examples)**  

---

### **What is YAML?**  
- **YAML** stands for **"YAML Ain't Markup Language"**.  
- It's a **human-readable** data format, used mainly for configuration files.  
- YAML uses **indentation** (spaces, not tabs) to represent data hierarchies.  

---

### **Why Use YAML?**  
- **Readable** – Easy to understand and write.  
- **Widely Used** – Common in tools like Kubernetes, Ansible, Docker Compose, etc.  
- **Flexible** – Supports key-value pairs, lists, and nested structures.  

---

### **YAML Basics:**  
#### **1. Key-Value Pairs (Mapping):**  
```yaml
name: John Doe
age: 30
city: New York
```

- **`name:`** is the key, **`John Doe`** is the value.  
- Use `:` followed by a space for assignment.  

---

#### **2. Lists (Arrays):**  
```yaml
skills:
  - Python
  - Docker
  - Kubernetes
```
- Lists start with a `-` followed by a space.  
- Indentation is crucial to show hierarchy.  

---

#### **3. Nested Structures (Hierarchy):**  
```yaml
person:
  name: John Doe
  age: 30
  address:
    city: New York
    country: USA
```
- Nested elements are indented.  

---

#### **4. Multi-line Strings:**  
```yaml
description: |
  This is a multi-line
  description.
  It preserves line breaks.
```

- **`|`** keeps line breaks.  
- **`>`** folds lines into one.  

---

#### **5. Comments:**  
```yaml
# This is a comment
title: YAML Tutorial  # Inline comment
```

---

### **Full Example – Kubernetes Pod Definition (YAML):**  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      ports:
        - containerPort: 80
```

---

### **Explanation (Step-by-Step):**  
1. **`apiVersion:`** – API version for Kubernetes resources (v1 for Pods).  
2. **`kind:`** – The type of Kubernetes resource (Pod in this case).  
3. **`metadata:`** – Contains Pod info like `name` and `labels`.  
4. **`spec:`** – Defines the Pod’s configuration.  
5. **`containers:`** – List of containers inside the Pod.  
   - **`image:`** – Docker image for the container.  
   - **`ports:`** – Opens port 80 inside the container.  

---

### **Running YAML Files in Kubernetes:**  
1. **Save the YAML file:**  
   ```bash
   vim pod.yaml
   ```  
2. **Deploy it:**  
   ```bash
   kubectl apply -f pod.yaml
   ```  
3. **Check status:**  
   ```bash
   kubectl get pods
   ```

---

### **Key YAML Rules:**  
- Use **spaces** for indentation (no tabs).  
- Indentation must be consistent.  
- Use `:` followed by a space for key-value pairs.  

---
