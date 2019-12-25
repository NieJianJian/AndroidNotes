## art/runtime/art_method.h文件

* Android环境：6.0

* [完整代码地址](https://www.androidos.net.cn/android/6.0.1_r16/xref/art/runtime/art_method.h)

* 简略结构如下：

  ```java
  class ArtMethod FINAL {
  	protected:
    	// Field order required by test "ValidateFieldOrderOfJavaCppUnionClasses".
    	// The class we are a part of.
    	GcRoot<mirror::Class> declaring_class_;
  
    	// Short cuts to declaring_class_->dex_cache_ member for fast compiled code access.
    	GcRoot<mirror::PointerArray> dex_cache_resolved_methods_;
  
    	// Short cuts to declaring_class_->dex_cache_ member for fast compiled code access.
    	GcRoot<mirror::ObjectArray<mirror::Class>> dex_cache_resolved_types_;
  
   	 // Access flags; low 16 bits are defined by spec.
    	uint32_t access_flags_;
  
   	 /* Dex file fields. The defining dex file is available via declaring_class_->dex_cache_ */
  
    	// Offset to the CodeItem.
    	uint32_t dex_code_item_offset_;
  
    	// Index into method_ids of the dex file associated with this method.
    	uint32_t dex_method_index_;
  
    	/* End of dex file fields. */
  
   	 	// Entry within a dispatch table for this method. For static/direct methods the index is into
    	// the declaringClass.directMethods, for virtual methods the vtable and for interface methods the
    	// ifTable.
    	uint32_t method_index_;
  
    	// Fake padding field gets inserted here.
  
    	// Must be the last fields in the method.
    	// PACKED(4) is necessary for the correctness of
    	// RoundUp(OFFSETOF_MEMBER(ArtMethod, ptr_sized_fields_), pointer_size).
    	struct PACKED(4) PtrSizedFields {
      	// Method dispatch from the interpreter invokes this pointer which may cause a bridge into
      	// compiled code.
      	void* entry_point_from_interpreter_;
  
      	// Pointer to JNI function registered to this method, or a function to resolve the JNI function.
      	void* entry_point_from_jni_;
  
      	// Method dispatch from quick compiled code invokes this pointer which may cause bridging into
      	// the interpreter.
      	void* entry_point_from_quick_compiled_code_;
    	} ptr_sized_fields_;
  }
  ```

  

