---
layout:     post
title:      "@BindView一行代码背后的故事-ButterKnife"
subtitle:   " \"Hello World, Hello Android\""
date:       2016-12-16 13:00:00
author:     "Cuieney"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 源码
---

> “Yeah It's on. ”


## 前提

>这篇文章呢主要讲的是ButterKnife IOC框架背后的故事，虽然网上很多这样的帖子，但是这篇细致到每个字段都会讲解（version=8.5.1，原理都一样可能版本不同，有些内部实行会有些不一样）就当埋点悬念吧。 @BindView一行代码到底给我做了哪些事情。这个框架就是为了给我们省去每次的findViewById这一行让你枯燥又乏味的代码块，到底他在后面都做了哪些故事呢！下面请听我侃侃道来...

### Annotation
哈哈哈哈上来就讲原理，不讲原理那怎么才能知道背后的故事啊，你说si不si啊！毕竟是个IOC框架 肯定要说到JAVA Annotation 这东西大家可以在日常的代码块经常看到的，这篇文章主要讲的是ButterKnife背后的故事呢，我这里就不会详细的解释Annotation，只说这个框架中用的一些。如果想了解Annotation呢可以参考一下这篇文章：
[传送门Annotation](http://trinea.github.io/download/pdf/android/java-annotation.pdf)

不论看没看这篇文章，我先说一下怎么自定义注解，了解各基本的大概就可以看懂这篇文章了。

#### 元注解（Retention，Target）
`@Rentention` 这个注解的意思是注解保留的时间，我们可以有以下三个选择

1.`SOURCE` 源码时保留，这类 Annotation 大都用来校验，比如 Override, Deprecated, SuppressWarnings

2.`CLASS` 肯定意思是编译时，就是我们在项目java文件在编译成class 的时候 apt 会自动解析 但需要做的是
    - 自定义类继承`AbstractProcessor`
    - 重写其中的`process`函数

这块可能会有同学不理解，实际是由apt在编译时自动查找所有继承来自`AbstractProcessor`的类，然后调用他们的process 方法去处理（我们这里的ButterKnife在这里就自定义了一个`ButterKnifeProcessor` 后面会详细讲解这个类）

3.`RUNTIME` 运行时保留，程序在运行过程中，使用这些 Annotation, 比如我们常用的 @Test。

`@Target`表示注解可以用来修饰哪些元素。可选值包括 TYPE, METHOD, CONSTRUCTOR, FIELD, PARAMETER 等

### ButterKnifeProcessor
由于我们的大神JakeWharton 每一个注解都是ClASS,所有java文件在编译的时候`ButterKnifeProcessor`的`process`就会被调用。好现在我们开始解析源码。

```
  @Override public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
  //这一行是根据env拿到所有带有相关注解根据TypeElement进行区分
    Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);
    //依次遍历生成相应的xxx_ViewBinding文件
    for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
      TypeElement typeElement = entry.getKey();
      BindingSet binding = entry.getValue();
        
      JavaFile javaFile = binding.brewJava(sdk);
      try {
        javaFile.writeTo(filer);
      } catch (IOException e) {
        error(typeElement, "Unable to write binding for type %s: %s", typeElement, e.getMessage());
      }
    }

    return false;
  }


```

上面的代码呢也不是太长，首选我们可以看到第一行创建了一个Map集合存放的key = TypeElement 而`TypeElement`是由`RoundEnvironment`通过
``` 
TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

```
如果不太明白Elements的意思
>作用：Elements是处理Element的工具类,Element代表程序的元素，例如包、类或者方法，可以理解成源代码;TypeElement代表的是源代码中的类型元素，例如类、域、方法等;从TypeElement中能获取类的名字，但是你获取不到类的信息，例如它的父类，这个需要从TypeMirror获取，而TypeMirror需要调用Element的asType()函数

value = BindingSet 这个类意思是什么呢。我们来看看源码啊 下面贴出的是`BindingSet`他的Builder

```
 static final class Builder {
    private final TypeName targetTypeName;
    private final ClassName bindingClassName;
    private final boolean isFinal;
    private final boolean isView;
    private final boolean isActivity;
    private final boolean isDialog;

    private BindingSet parentBinding;
    //存储（@BindView（id））这个id的
    private final Map<Id, ViewBinding.Builder> viewIdMap = new LinkedHashMap<>();
    private final ImmutableList.Builder<FieldCollectionViewBinding> collectionBindings =
        ImmutableList.builder();
    private final ImmutableList.Builder<ResourceBinding> resourceBindings = ImmutableList.builder();

    private Builder(TypeName targetTypeName, ClassName bindingClassName, boolean isFinal,
        boolean isView, boolean isActivity, boolean isDialog) {
      this.targetTypeName = targetTypeName;
      this.bindingClassName = bindingClassName;
      this.isFinal = isFinal;
      this.isView = isView;
      this.isActivity = isActivity;
      this.isDialog = isDialog;
    }


```
为什么贴出他的Builder呢，因为这样更容易理解这个类干嘛的，他是保存一个类（当前的Activity）里面到底有哪些关于ButterKnife的注解。上面的viewIdMap就是用于存储（@BindView（id））这个id的，我们在看看Builder这个内部类的一些方法可能你会更理解他到底在做哪些事情
```
 //用于@BindView(R.id.test)
 void addField(Id id, FieldViewBinding binding) {
      getOrCreateViewBindings(id).setFieldBinding(binding);
    }

    void addFieldCollection(FieldCollectionViewBinding binding) {
      collectionBindings.add(binding);
    }

    //方法的bind
    boolean addMethod(
        Id id,
        ListenerClass listener,
        ListenerMethod method,
        MethodViewBinding binding) {
      ViewBinding.Builder viewBinding = getOrCreateViewBindings(id);
      if (viewBinding.hasMethodBinding(listener, method) && !"void".equals(method.returnType())) {
        return false;
      }
      viewBinding.addMethodBinding(listener, method, binding);
      return true;
    }
    //用于@BindBitmap @BindDimen...就是一些资源文件的bind
    void addResource(ResourceBinding binding) {
      resourceBindings.add(binding);
    }

```
从上面的代码可以看到这个类`BuilderSet`到底干了些什么事吧，就是把你添加注释的这个类的信息保存下来，后面做判断，做代码的生成。

说了这么多其实就是解释`process()`第一行Map代码到底是做什么的，接下来我们看`process（）`里面的循环到底干什么的。上面的代码块我也写了一些注释，说是生成对应的`xxx_ViewBinding`文件的。如何生成的呢？细心的同学会注意到那个里面的`filer`这个东西，其实这个是在我们初始化的时候的一些工具,下面是ButterKnife初始化的的一些操作
```
@Override public synchronized void init(ProcessingEnvironment env) {
    super.init(env);

    String sdk = env.getOptions().get(OPTION_SDK_INT);
    if (sdk != null) {
      try {
        this.sdk = Integer.parseInt(sdk);
      } catch (NumberFormatException e) {
        env.getMessager()
            .printMessage(Kind.WARNING, "Unable to parse supplied minSdk option '"
                + sdk
                + "'. Falling back to API 1 support.");
      }
    }
    //scan java文件每一个Element
    elementUtils = env.getElementUtils();
    //是用来处理TypeMirror的工具类
    typeUtils = env.getTypeUtils();
    //用来创建生成辅助文件
    filer = env.getFiler();
    try {
      trees = Trees.instance(processingEnv);
    } catch (IllegalArgumentException ignored) {
    }
  }

```
就是一些初始化操作。主要就elementUtils，typeUtils，filer这个三个工具的初始化，具体干嘛的上面代码我已经写了注释了。

这个先告一段落（具体如何生成的我后面会讲到）。我们知道在java 文件编译的时候`ButterKnifeProcessor`靠着`process()`这个方法生成了队友的`xxx_ViewBinding`文件。***那么问题来了***，我们如何把这个文件和我们的添加了注解的文件（xxxActivity.java,后面就用xx代替了）绑定在一起呢。

### 如何绑定xxx_ViewBinding
相信大家用过BindKnife的人都知道，要在我们的BaseActivity里面或者当前的Activity中bind（setContentView或者OnViewCreated之后做这个操作） 和 unBind一下。这个就是关键。这里就拿`@BindView`做列举。废话不多说上代码
```
  @NonNull @UiThread
  public static Unbinder bind(@NonNull Activity target) {
    //获取最外层View
    View sourceView = target.getWindow().getDecorView();
    return createBinding(target, sourceView);
  }

```
下面的是上方代码createBinding的具体实现
```
private static Unbinder createBinding(@NonNull Object target, @NonNull View source) {
    //获取当前这个类
    Class<?> targetClass = target.getClass();
    if (debug) Log.d(TAG, "Looking up binding for " + targetClass.getName());
    //通过这个类然后找到对于的xxx_ViewBinding文件的构造方法
    Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);

    if (constructor == null) {
      return Unbinder.EMPTY;
    }

    //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
    try {
        //初始化这个xxx_ViewBinding文件
      return constructor.newInstance(target, source);
    } catch (IllegalAccessException e) {
      ....
    }
  }

```
上面的代码我已经写了注释了，可以看到最主要的代码是`findBindingConstructorForClass`这个方法找到我们的这个当前的这个Activity对于的xxx_ViewBinding 然后获取他的构造方法，然后初始，那我们进入这个方法看看到底做了哪些操作。
```
 @Nullable @CheckResult @UiThread
  private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {
  //从集合中获取这个xxx_ViewBinding的构造函数（这个map用于缓存用下次就不需要下面的操作来获取了）
    Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
    if (bindingCtor != null) {
      if (debug) Log.d(TAG, "HIT: Cached in binding map.");
      return bindingCtor;
    }
    //获取clsName
    String clsName = cls.getName();
    //过滤不需要的
    if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
      if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
      return null;
    }
    try {
    //通过反射获取这个xxx_ViewBinding的class然后获取他的构造函数
    //细心的同学可以看到这里面接受了两个参数，一个是这个cls的父类和当前最外层的view
      Class<?> bindingClass = Class.forName(clsName + "_ViewBinding");
      //noinspection unchecked
      bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
      if (debug) Log.d(TAG, "HIT: Loaded binding class and constructor.");
    } catch (ClassNotFoundException e) {
      if (debug) Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
      bindingCtor = findBindingConstructorForClass(cls.getSuperclass());
    } catch (NoSuchMethodException e) {
      throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
    }
    //上面如果没有从map集合中获取，通过反射回去的会添加到集合中方便下次直接获取。就是缓存的意思
    BINDINGS.put(cls, bindingCtor);
    return bindingCtor;
  }

```

上面的代码看到了吗？每行的注释都有，可以看到他是通过反射的方式拿到这个xxx_ViewBinding文件然后获取构造他的构造方法的。然后通过BINDINGS这个集合来做缓存，减少耗时操作毕竟用反射都很耗时的。


接下来我们来看看生成的到底是一个什么样的文件xxx_ViewBinding


```
public class CameraActivityRep_ViewBinding implements Unbinder {
  private CameraActivityRep target;

  @UiThread
  public CameraActivityRep_ViewBinding(CameraActivityRep target) {
    this(target, target.getWindow().getDecorView());
  }

  @UiThread
  public CameraActivityRep_ViewBinding(CameraActivityRep target, View source) {
    this.target = target;
    //就是findviewById
    target.modelPanorama = Utils.findRequiredViewAsType(source, R.id.model_panorama, "field 'modelPanorama'", ImageView.class);
    target.modelCapture = Utils.findRequiredViewAsType(source, R.id.model_capture, "field 'modelCapture'", ImageView.class);
   
  }

  @Override
  @CallSuper
  public void unbind() {
    CameraActivityRep target = this.target;
    if (target == null) throw new IllegalStateException("Bindings already cleared.");
    this.target = null;

    target.modelPanorama = null;
    target.modelCapture = null;
   
  }

```
看到这里我们终于看到了我们的findViewById在哪里了在他的生成文件的构造函数中进行的findViewById，ButterKnife.bind(this);这个的作用就是findViewById的作用，通过bind的方法获取生成的xxx_ViewBinding文件，然后通过反射获取构造函数，到构造函数的初始化。在构造函数里面做了findViewById的操作。

其实大伙可能说我明明没看到findViewById就看到了`Utils.findRequiredViewAsType(source, R.id.model_panorama, "field 'modelPanorama'", ImageView.class)`这行代码，好我们接下来继续看这个utils到底干了啥是不是findViewById

```
 public static <T> T findRequiredViewAsType(View source, @IdRes int id, String who,
      Class<T> cls) {
      //MD我咋还没看到呢继续往下看
    View view = findRequiredView(source, id, who);
    return castView(view, id, who, cls);
  }

```
MD我咋还没看到呢继续往下看

```

 public static View findRequiredView(View source, @IdRes int id, String who) {
    //看到了吗 看到了吧
    View view = source.findViewById(id);
    if (view != null) {
      return view;
    }
    String name = getResourceEntryName(source, id);
    throw new IllegalStateException("Required view '"
        + name
        + "' with ID "
        + id
        + " for "
        + who
        + " was not found. If this view is optional add '@Nullable' (fields) or '@Optional'"
        + " (methods) annotation.");
  }

```
好了同学们知道了吧，小伙子隐藏的可真深啊。

~~**mdzz 一句findViewById 引发了这么多东西 这就是@BindView背后不可告知的秘密**~~有兴趣的同学可以接着往下读，看看他是如何生成xxx_ViewBinding的

### 如何生成xxx_ViewBinding
我们继续回讲一下刚刚的ButterKnifeProcessor那个`process（）`这个方法不知道还记不记得里面的代码我们就在贴一遍吧
```
  @Override public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
  //这一行是根据env拿到所有带有相关注解根据TypeElement进行区分
    Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);
    //依次遍历生成相应的xxx_ViewBinding文件
    for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
      TypeElement typeElement = entry.getKey();
      BindingSet binding = entry.getValue();
        
      JavaFile javaFile = binding.brewJava(sdk);
      try {
        javaFile.writeTo(filer);
      } catch (IOException e) {
        error(typeElement, "Unable to write binding for type %s: %s", typeElement, e.getMessage());
      }
    }

    return false;
  }


```

由于上面已经讲了这个方法里面的一些参数东西，我这里就不重复了，之说一些关键点

1.获取带有注解的所有Element然后把每个TypeElement对应的BindingSet一一对应存储在Map中
```
findAndParseTargets(env)

```

2.生成对应得xxx_ViewBinding文件
```
JavaFile javaFile = binding.brewJava(sdk);
javaFile.writeTo(filer);

```
我们可以看到这两点，我们先说第一个吧。既然是方法，肯定要往方法里面走了，看看源码在做一些什么东西。

```
private Map<TypeElement, BindingSet> findAndParseTargets(RoundEnvironment env) {
    Map<TypeElement, BindingSet.Builder> builderMap = new LinkedHashMap<>();
    Set<TypeElement> erasedTargetNames = new LinkedHashSet<>();

    scanForRClasses(env);

    ......
    
    // 找到每个带有 @BindView element 添加到集合中.
    for (Element element : env.getElementsAnnotatedWith(BindView.class)) {
      // we don't SuperficialValidation.validateElement(element)
      // so that an unresolved View type can be generated by later processing rounds
      try {
        parseBindView(element, builderMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindView.class, e);
      }
    
    
    .......
    
    // 就是把一个<TypeElement, BindingSet.Builder> -> TypeElement, BindingSet
    //其实就是把一个activity的所有带有@bind的注解存在在BindingSet中
    //然后返回给process（）加工成文件
    Deque<Map.Entry<TypeElement, BindingSet.Builder>> entries =
        new ArrayDeque<>(builderMap.entrySet());
    Map<TypeElement, BindingSet> bindingMap = new LinkedHashMap<>();
    while (!entries.isEmpty()) {
      Map.Entry<TypeElement, BindingSet.Builder> entry = entries.removeFirst();

      TypeElement type = entry.getKey();
      BindingSet.Builder builder = entry.getValue();

      TypeElement parentType = findParentType(type, erasedTargetNames);
      if (parentType == null) {
        bindingMap.put(type, builder.build());
      } else {
        BindingSet parentBinding = bindingMap.get(parentType);
        if (parentBinding != null) {
          builder.setParent(parentBinding);
          bindingMap.put(type, builder.build());
        } else {
          // Has a superclass binding but we haven't built it yet. Re-enqueue for later.
          entries.addLast(entry);
        }
      }
    }

    return bindingMap;
  }

```
这里代码其实比这长多了，这里我就拿BindView注解讲吧，其他的都类似的。把其他的都删了不然代码实在太长，我们可以看到上面的代码通过`env.getElementsAnnotatedWith(BindView.class)`找到带有@BindView的element然后遍历循环，然后接下来他通过一个方法`parseBindView`把这些Element做了一个些整理就是把一个Acitvity里面的所有注解对应起来。我们来看看到底做了那些事情

```
private void parseBindView(Element element, Map<TypeElement, BindingSet.Builder> builderMap,
      Set<TypeElement> erasedTargetNames) {
      //这句代码的意思就是获取拿到一个标识（xxxAcitivty的意思）
    TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();
    
    ....
    
    // 获取@BindView（R.id.test）获取这个id的
    int id = element.getAnnotation(BindView.class).value();
    //拿到这个标识对应的BindingSet，在BindingSet里面有个map存这个act里面有多少@bindview注解
    BindingSet.Builder builder = builderMap.get(enclosingElement);
    if (builder != null) {
      String existingBindingName = builder.findExistingBindingName(getId(id));
      //如果发现这个id已经存进去了，直接return
      if (existingBindingName != null) {
        error(element, "Attempt to use @%s for an already bound ID %d on '%s'. (%s.%s)",
            BindView.class.getSimpleName(), id, existingBindingName,
            enclosingElement.getQualifiedName(), element.getSimpleName());
        return;
      }
    } else {
    //发现buildSet 的map中并没有存这个id 那我们就把他添加进去
      builder = getOrCreateBindingBuilder(builderMap, enclosingElement);
    }

    String name = element.getSimpleName().toString();
    TypeName type = TypeName.get(elementType);
    boolean required = isFieldRequired(element);

    builder.addField(getId(id), new FieldViewBinding(name, type, required));

    // Add the type-erased version to the valid binding targets set.
    erasedTargetNames.add(enclosingElement);
  }

```
上面解释也写了很多，我这边就大概的讲一下这个方法干嘛的，首先呢我们拿到传进来的builderMap，这个map对应的是key = TypeElement（相当于当前Act的一个标识） value = BindingSet.Builder（存储着这个Act里面的所有注解），然后我们根据传进来的element 到map中查找看看这个Element对应的BuildSet里面是否包含这个id，如果包含了直接返回，没有的话拿到这个Element对应的BuildSet 往里面添加这个id（通过 builder.addField）

这个就是BindView干的一些事情。这里就在总结一下上面的东西
- 每一个TypeElement相当于一个（Activity，fragment，dialog）
- 每一个BindingSet存储了TypeElement里面所有包含注解的信息

这里就告一段落了，那我们看看代码的生成，拿到BindSet生成对于的xxx_ViewBinding文件
```
binding.brewJava(sdk).writeTo(filer)

```


```
  JavaFile brewJava(int sdk) {
    return JavaFile.builder(bindingClassName.packageName(), createType(sdk))
    //顾名思义添加注释的意思
        .addFileComment("Generated code from Butter Knife. Do not modify!")
        .build();
  }

```

```
  public void writeTo(Filer filer) throws IOException {
    String fileName = packageName.isEmpty()
        ? typeSpec.name
        : packageName + "." + typeSpec.name;
    List<Element> originatingElements = typeSpec.originatingElements;
    JavaFileObject filerSourceFile = filer.createSourceFile(fileName,
        originatingElements.toArray(new Element[originatingElements.size()]));
    try (Writer writer = filerSourceFile.openWriter()) {
      writeTo(writer);
    } catch (Exception e) {
      try {
        filerSourceFile.delete();
      } catch (Exception ignored) {
      }
      throw e;
    }
  }

```
这里面代码也挺多了我就不一一进去讲解了，这里用的是[javapoet](https://github.com/square/javapoet)来进行代码的写入的，感兴趣的同学可以看一看多艺技不压身。

### ending
卧槽写完了咋感觉头懵懵的，但是还是希望这篇文章带给你的是知识的提升而不是时间的浪费（毕竟写了几小时呢）

—— Cuieney 后记于 2017.03


