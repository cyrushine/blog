# coroutine vs thread

`thread {...}` 创建一个新线程执行 block 并立刻返回，它是 non-blocking 的，`CoroutineScope.launch` 也一样不过它创建的是 `Continuation/Job`

```kotlin
CoroutineScope.launch
AbstractCoroutine.start // StandaloneCoroutine
CoroutineStart.invoke(block, receiver, completion) // CoroutineStart.DEFAULT
startCoroutineCancellable(receiver, completion, onCancellation)

```

```kotlin
// https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/jvm/test/guide/example-basic-01.kt
fun main() = runBlocking { // this: CoroutineScope
    launch { // launch a new coroutine and continue
        delay(1000L) // non-blocking delay for 1 second (default time unit is ms)
        println("World!") // print after delay
    }
    println("Hello") // main coroutine continues while a previous one is delayed
}
```

decompile by [bytecode-viewer](https://github.com/Konloch/bytecode-viewer)

lamda 参数 `block: suspend CoroutineScope.() -> Unit` 被编译为继承自 `SuspendLamda` 和 `Function2<CoroutineScope, Continuation>`

```java
final class Example_basic_01Kt$main$1 extends SuspendLambda implements Function2 {
   int label;
   // $FF: synthetic field
   private Object L$0;

   Example_basic_01Kt$main$1(Continuation $completion) {
      super(2, $completion);
   }

   @Nullable
   public final Object invokeSuspend(@NotNull Object var1) {
      Object var5 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
      switch(this.label) {
      case 0:
         ResultKt.throwOnFailure(var1);
         CoroutineScope $this$runBlocking = (CoroutineScope)this.L$0;
         BuildersKt.launch$default($this$runBlocking, (CoroutineContext)null, (CoroutineStart)null, (Function2)(new 1((Continuation)null)), 3, (Object)null);
         String var3 = "Hello";
         boolean var4 = false;
         System.out.println(var3);
         return Unit.INSTANCE;
      default:
         throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
      }
   }

   @NotNull
   public final Continuation create(@Nullable Object value, @NotNull Continuation $completion) {
      Example_basic_01Kt$main$1 var3 = new Example_basic_01Kt$main$1($completion);
      var3.L$0 = value;
      return (Continuation)var3;
   }

   @Nullable
   public final Object invoke(@NotNull CoroutineScope p1, @Nullable Continuation p2) {
      return ((Example_basic_01Kt$main$1)this.create(p1, p2)).invokeSuspend(Unit.INSTANCE);
   }
}

final class Example_basic_01Kt$main$1$1 extends SuspendLambda implements Function2 {
   int label;

   Example_basic_01Kt$main$1$1(Continuation $completion) {
      super(2, $completion);
   }

   @Nullable
   public final Object invokeSuspend(@NotNull Object $result) {
      Object var4 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
      switch(this.label) {
      case 0:
         ResultKt.throwOnFailure($result);
         Continuation var10001 = (Continuation)this;
         this.label = 1;
         if (DelayKt.delay(1000L, var10001) == var4) {
            return var4;
         }
         break;
      case 1:
         ResultKt.throwOnFailure($result);
         break;
      default:
         throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
      }

      String var2 = "World!";
      boolean var3 = false;
      System.out.println(var2);
      return Unit.INSTANCE;
   }

   @NotNull
   public final Continuation create(@Nullable Object value, @NotNull Continuation $completion) {
      return (Continuation)(new Example_basic_01Kt$main$1$1($completion));
   }

   @Nullable
   public final Object invoke(@NotNull CoroutineScope p1, @Nullable Continuation p2) {
      return ((Example_basic_01Kt$main$1$1)this.create(p1, p2)).invokeSuspend(Unit.INSTANCE);
   }
}

```