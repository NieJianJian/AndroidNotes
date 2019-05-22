## java注解



* 限制参数的值和范围

  ```java
      public static final int VERTICAL = 0;
      public static final int HORIZONTAL = 1;
  
      @IntDef({VERTICAL, HORIZONTAL})
      public @interface OrientationType {
      }
  
      /**
       * @param orientation 只能输入 VERTICAL 和 HORIZONTAL
       * @param count 只能输入0-10的值。
       */
      public static void setOrientation(@OrientationType int orientation,
                                        @IntRange(from = 0, to = 10) int count) {
          // do something...
      }
  ```

  

