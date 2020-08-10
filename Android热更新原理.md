# 热更新

## ClassLoader

1. `PathClassLoader`: 只能加载系统中已经安装过的apk
2. `DexClassLoader`: 加载jar/apk/dex，可以从SD卡中加载未安装的apk

他们的基类都是 `BaseDexClassLoader`

## 代码热更新

### BaseDexClassLoader

#### 构造函数

1. `dexPath`: apk/jar/dex文件 或 包含这些文件的文件夹
2. `optimizedDirectory`: >= 26 已被废弃, < 26是
3. `librarySearchPath`: 包含native库的文件夹
4. `dexFiles`: 已经加载到内存中的dex文件

#### findClass

Java是通过调用`ClassLoader.loadClass`(这里面进行了查找缓存和双亲委派) -> `ClassLoader.findClass`将类加载到虚拟机的, 而`BaseDexClassLoader.findClass`又调用了`DexPathList.findClass`去dex文件中查找对应的class文件(通过遍历`DexPathList.dexElements`), 我们只要将热更新后的dex放到数组首位就能优先加载热更新后的代码了

```Java
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            ClassNotFoundException cnfe = new ClassNotFoundException(
                    "Didn't find class \"" + name + "\" on path: " + pathList);
            for (Throwable t : suppressedExceptions) {
                cnfe.addSuppressed(t);
            }
            throw cnfe;
        }
        return c;
    }
```

#### DexPathList

```Java

    //根据传入的类名, 查找在哪个dex中
    public Class<?> findClass(String name, List<Throwable> suppressed) {
        for (Element element : dexElements) {
            Class<?> clazz = element.findClass(name, definingContext, suppressed);
            if (clazz != null) {
                return clazz;
            }
        }

        if (dexElementsSuppressedExceptions != null) {
            suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
        }
        return null;
    }


    //遍历搜索所有的dex文件(也包含在压缩包里的)
    private static Element[] makeDexElements(List<File> files, File optimizedDirectory,
            List<IOException> suppressedExceptions, ClassLoader loader, boolean isTrusted) {
      Element[] elements = new Element[files.size()];
      int elementsPos = 0;
      /*
       * Open all files and load the (direct or contained) dex files up front.
       */
      for (File file : files) {
          if (file.isDirectory()) {
              // We support directories for looking up resources. Looking up resources in
              // directories is useful for running libcore tests.
              elements[elementsPos++] = new Element(file);
          } else if (file.isFile()) {
              String name = file.getName();

              DexFile dex = null;
              //如果直接是dex文件
              if (name.endsWith(DEX_SUFFIX)) {
                  // Raw dex file (not inside a zip/jar).
                  try {
                      dex = loadDexFile(file, optimizedDirectory, loader, elements);
                      if (dex != null) {
                          elements[elementsPos++] = new Element(dex, null);
                      }
                  } catch (IOException suppressed) {
                      System.logE("Unable to load dex file: " + file, suppressed);
                      suppressedExceptions.add(suppressed);
                  }
              } else {
                  try {
                      //有可能压缩文件里只有资源, 没有dex文件, 会抛异常
                      dex = loadDexFile(file, optimizedDirectory, loader, elements);
                  } catch (IOException suppressed) {
                      /*
                       * IOException might get thrown "legitimately" by the DexFile constructor if
                       * the zip file turns out to be resource-only (that is, no classes.dex file
                       * in it).
                       * Let dex == null and hang on to the exception to add to the tea-leaves for
                       * when findClass returns null.
                       */
                      suppressedExceptions.add(suppressed);
                  }

                  if (dex == null) {
                      elements[elementsPos++] = new Element(file);
                  } else {
                      elements[elementsPos++] = new Element(dex, file);
                  }
              }
              if (dex != null && isTrusted) {
                dex.setTrusted();
              }
          } else {
              System.logW("ClassLoader referenced unknown path: " + file);
          }
      }
      if (elementsPos != elements.length) {
          elements = Arrays.copyOf(elements, elementsPos);
      }
      return elements;
    }
```

#### demo

```kotlin
    private fun hotfix() {
        val classLoader = classLoader as PathClassLoader
        val dexPathListClass = Class.forName("dalvik.system.DexPathList")
        //这个方法能传入文件列表, 生成 Element数组
        //makeDexElements(List<File> files, File optimizedDirectory, List<IOException> suppressedExceptions, ClassLoader loader)
        val makeDexElementsMethod =
            dexPathListClass.getDeclaredMethod(
                "makeDexElements",
                List::class.java,
                File::class.java,
                List::class.java,
                ClassLoader::class.java
            ).apply { isAccessible = true }
        //热更新文件, 可以是.jar, .dex .zip .apk
        val apk = File(externalCacheDir, "new.apk")
        if (apk.exists()) {
            //拿到新 Element数组
            val newElements =
                makeDexElementsMethod.invoke(
                    null,
                    listOf(apk),
                    null,
                    mutableListOf<IOException>(),
                    classLoader
                ) as kotlin.Array<Any>
            //拿到 PathClassLoader的旧数据
            val pathListFiled = BaseDexClassLoader::class.java.getDeclaredField("pathList")
                .apply { isAccessible = true }
            val pathList = pathListFiled.get(classLoader)
            val elementsFiled =
                dexPathListClass.getDeclaredField("dexElements").apply { isAccessible = true }
            //旧数组
            val oldElements = elementsFiled.get(pathList) as kotlin.Array<Any>
            val elementClass = oldElements::class.java.componentType
            //生成新数组
            val hotfixedElements = Array.newInstance(
                elementClass,
                oldElements.size + newElements.size
            ) as kotlin.Array<Any>
            //把新旧 Element都放进去, 优先加载新的
            var index = 0
            for (i in newElements.indices) {
                hotfixedElements[index++] = newElements[i]
            }
            for (i in oldElements.indices) {
                hotfixedElements[index++] = oldElements[i]
            }
            //替换原来的对象
            elementsFiled.set(pathList, hotfixedElements)
        }
    }
```

## 资源热更新