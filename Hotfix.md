### AssetBundle Lua热更流程

1. 要热更的资源标记AssetBundle，点击Asset -> Build AssetBundles 按钮打包，打包后发现项目根目录中的AssetBundles 文件夹, 里面的就是我们打包出来的AssetBundle资源，最终会传到服务器上

2. 通过服务器下载热更资源到AB包

   用AB包load资源：

   ```csharp
   var luaScript = assetBundle.LoadAsset<TextAsset>("luaScript.lua.txt");
   ```

   将lua脚本写入到

   ```csharp
   Application.persistentDataPath + @"/luaScript.lua.txt"
   ```

3. 执行热更脚本：
   ```csharp
   LuaEnv luaenv = new LuaEnv();
   luaenv.AddLoader(MyLoader);
   luaenv.DoString("require 'luaScript'");
   ```

   自定义的loader是指定了load的路径为Application.persistentDataPath，否则就是用默认的Resources

### Lua配置

**[CSharpCallLua]**：在c#读取lua的数据时，在c#端应转换为的类型
lua table->interface；lua function->自定义delegate要加
**[LuaCallCSharp]**：在lua读取c#类型时，在c#端的类型要加，生成适配的代码
xLua会生成这个类型的适配代码（包括构造该类型实例，访问其成员属性、方法，静态属性、方法），不会自动生成其父类的适配代码，如果该父类加了LuaCallCSharp配置，则执行父类的适配代码，否则会尝试用反射来访问
有在lua用到的都要加
https://github.com/Tencent/xLua/blob/master/Assets/XLua/Doc/configure.md

### C#与Lua交互

需要GenerateCode：类新增了public方法 lua端要调用
新增hotfix时
需要HotfixInject：修改了标记了[Hotfix]的类里面的内容（不用generate 只用reject）

#### C#调用Lua

1. 基本数据类型

   ```csharp
    luaenv.Global.Get<int>("a")  // lua中 a = 1
   ```

2. table

   通过值拷贝

   i. 映射到普通class或struct（值拷贝，按字段名称，table和class字段数量可以不同）

   ```csharp
   public class DClass
   {
       public int f1;
       public int f2;
   }
   luaenv.Global.Get<DClass>("d").f1
   ```

   ```lua
   d = {
      f1 = 12, f2 = 34,
      1, 2, 3,
      add = function(self, a, b) 
         return a + b 
      end
   }
   ```

   ii. 映射到Dictionary<string,>，List<>

   ```csharp
   // 只有f1 f2
   luaenv.Global.Get<Dictionary<string, int>>("d");
   // 只有3个元素1, 2, 3
   luaenv.Global.Get<List<int>>("d");
   ```

   通过引用：

   iii. 映射到interface（引用，要生成代码，get接口属性时，会查找table对应字段）

   ```csharp
   [CSharpCallLua]
   public interface DInterface
   {
       int f1 { get; set; }
       int f2 { get; set; }
       int add(int a, int b);
   }
   luaenv.Global.Get<DInterface>("d")
   ```

   iv. 映射到LuaTable类（不用生成代码，但是比interface慢很多）

   ```csharp
   var tableD = luaenv.Global.Get<LuaTable>("d");
   tableD.Get<int>("f1")
   ```

3. function

   i. 映射到delegate

   输入参数：输入参数、ref

   多返回值：第1个是return 之后按顺序out、ref

   ```cs
   // Action不用生成代码
   var actionE = luaenv.Global.Get<Action>("e");
   ```

   ```csharp
   // 自定义delegate类型要生成代码
   [CSharpCallLua]
   public delegate int FunF(int a, int b, out DClass f);
   
   luaenv.Global.Get<FunF>("f");
   ```

   ```lua
   function f(a, b)
       return 1, {f1 = 1024}
   end
   ```

   ii. 映射到LuaFunction（不用生成代码 但是性能不好）

   ```csharp
   luaenv.Global.Get<LuaFunction>("f");
   ```

#### Lua调用C#

1. C#对象

   用CS.前缀

   ```lua
   local go1 = CS.UnityEngine.GameObject()
   ```

2. 对象成员变量、方法

   ```lua
   local obj = CS.DerivedClass()
   obj.DMF = 1024
   obj:Func()
   --还可以把常用的类型先缓存：
   local DerivedClass = CS.DerivedClass
   local obj = DerivedClass()
   ```

   ```csharp
   [LuaCallCSharp]
   public class DerivedClass
   {
       public int DMF;
       public void Func()
       {
           Debug.Log($"DMF:{DMF}");
       }
   }
   ```

3. 基类属性，方法（和c#一样直接调就行）

   ```lua
   obj.DSF = 111
   ```

   ```csharp
   [LuaCallCSharp]
   public class DerivedClass : BassClass
   {
   }
   [LuaCallCSharp]
   public class BassClass
   {
       public int DSF;
   }
   ```

4. 复杂方法调用

   lua这边：参数：c#方法的普通参数，ref；返回值：c#方法的返回值，out，ref，按顺序对应

   ```lua
   local res1, p2, p3, outfunc = obj:ComplexFunc(
       { x = 1, y = 'john'},
       10,
       callback)
       outfunc()
   ```

   ```csharp
   [LuaCallCSharp]
   public class DerivedClass
   {
       public double ComplexFunc(Param1 p1, ref int p2, out string p3, Action luafunc, out Action csfunc)
       {
          	...
       }
   }
   public struct Param1
   {
       public int x;
       public string y;
   }
   ```

5. 重载方法

   和c#一样，但是c#的int float都对应lua的number，这种情况只能调用到生成代码中前面那一个

6. 操作符（和C#一样，支持：+，-，*，/，==，一元-，<，<=， %，[]）

7. 枚举

   ```lua
    --从整数或字符串转换为枚举值（从0开始）
   print(CS.TestEnum.__CastFrom(1), CS.TestEnum.__CastFrom('E1'))
   --true 因为这个枚举定义在子类 和c#一样的
   print(CS.BaseClass.TestEnumInner == nil) 
   ```

   ```csharp
   [LuaCallCSharp]
   public enum TestEnum {E1, E2}
   [LuaCallCSharp]
   public class DerivedClass : BassClass
   {
       [LuaCallCSharp]
       public enum TestEnumInner {E3, E4}
   }
   ```

8. 注册事件（event）

   ```lua
   obj:TestEvent('+', callback);
   obj:InvokeTestEvent(); --间接调用，lua属于外部调用者不能invoke event
   obj:TestEvent('-', callback);
   ```

   ```csharp
   [LuaCallCSharp]
   public class DerivedClass
   {
       public event Action TestEvent;
       public void InvokeTestEvent()
       {
           TestEvent?.Invoke();
       }
   }
   ```

9. 注册委托（注意是.不是:）

   ```lua
   obj.TestDelegate = obj.TestDelegate + callback;
   obj.TestDelegate(); --直接调用
   obj.TestDelegate = obj.TestDelegate - callback;
   ```

   ```csharp
   [LuaCallCSharp]
   public class DerivedClass
   {
       public Action TestDelegate;
   }
   ```

10. 嵌套结构

    ```lua
    obj:Foo({a = {a = 10}, b = 20});
    ```

    ```csharp
    [LuaCallCSharp]
    public class DerivedClass
    {
        public void Foo(B b) {}
    }
    public struct A
    {
        public int a;
    }
    public struct B
    {
        public A a;
        public int b;
    }
    ```

11. 接口或抽象类

    lua端获得的是接口或抽象类时，lua无法对实现类进行代码生成，就会用反射的方式调用（慢），所以要用cast将接口或抽象类进行代码生成

    ```lua
    local calc = obj:GetCalc();
    cast(calc, typeof(CS.ICalc));
    calc:Get();
    ```

    ```csharp
    public ICalc GetCalc()
    {
        return new InnerCalc();
    }
    public interface ICalc
    {
        int Get();
    }
    ```

### Lua热更新

简单说就是luaenv运行xlua.hotfix将代码替换
**xlua.hotfix(class, [method_name], fix)**
    描述 ： 注入lua补丁
    class ： C#类，两种表示方法，CS.Namespace.TypeName或者字符串方式"Namespace.TypeName"，字符串格式和C#的Type.GetType要求一致，如果是内嵌类型（Nested Type）是非Public类型的话，只能用字符串方式表示"Namespace.TypeName+NestedTypeName"；
    method_name ： 方法名，可选；
    fix ： 如果传了method_name，fix将会是一个function，否则通过table提供一组函数。table的组织按key是method_name，value是function的方式。