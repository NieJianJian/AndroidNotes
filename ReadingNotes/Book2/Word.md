* **CLASS_ISPREVERIFIED**：如果某类的所有方法中直接引用到的类（第一层关系，不会进行递归搜索）和当前类都在同一个dex中的话，dvmVerifyClass就会返回true，然后这个类就会被打上CLASS_ISPREVERIFIED标签。被打上该标签的类只能引用同一个dex中的类，否则就会报错。

