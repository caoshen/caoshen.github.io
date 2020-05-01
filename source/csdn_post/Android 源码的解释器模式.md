# Android 源码的解释器模式

## 解释器模式介绍

解释器模式是一种用得比较少的行为型模式，其提供了一种解释语言的语法或表达式的方式。该模式定义了一个表达式接口，通过该接口解释一个特定的上下文。

## 解释器模式的定义

给定一个语言，定义它的文法的一种表示，并定义一个解释器，该解释器使用该表示来解释语言中的句子。

## PackageParser

Android 的 PackageParser 类对 AndroidManifest.xml 中的每一个组件标签创建了相应的类用以存储相应的信息。

```java
    public final static class Activity extends Component<ActivityIntentInfo> implements Parcelable {
    ...
    public final static class Service extends Component<ServiceIntentInfo> implements Parcelable {
    ...
    public final static class Provider extends Component<ProviderIntentInfo> implements Parcelable {
    ...
    public final static class Instrumentation extends Component<IntentInfo> implements
    ...
```

PackageParser 为 Activity、Service、Provider 等组件在其内部以内部类的方式创建了对应的类，按照解释器模式的定义，这些类都对应 AndroidManifest.xml 文件中的一个标签，也就是一条文法，其在对该配置文件解析时充分运用了解释器模式分离实现、解释执行的特性。

在 Android 中，解析某个 apk 文件会调用到 PackageManagerService 中的 scanPackageLI 方法。

scanPackageLI 方法会调用 PackageParser 的 parsePackage 方法。

```java
    private PackageParser.Package scanPackageLI(File scanFile, int parseFlags, int scanFlags,
            long currentTime, UserHandle user) throws PackageManagerException {
        PackageParser pp = new PackageParser();
        ...
        final PackageParser.Package pkg;
        try {
            pkg = pp.parsePackage(scanFile, parseFlags);
        }
        ...
        return scanPackageChildLI(pkg, parseFlags, scanFlags, currentTime, user);
    }
```

parsePackage 方法经过多次调用，最后会调用 parseBaseApkCommon 方法，解析 AndroidManifest 的每个子节点。

```java
    private Package parseBaseApkCommon(Package pkg, Set<String> acceptedTags, Resources res,
            XmlResourceParser parser, int flags, String[] outError) throws XmlPullParserException,
            IOException {
            ...
            if (tagName.equals(TAG_APPLICATION)) {
                if (foundApp) {
                    if (RIGID_PARSER) {
                        outError[0] = "<manifest> has more than one <application>";
                        mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                        return null;
                    } else {
                        Slog.w(TAG, "<manifest> has more than one <application>");
                        XmlUtils.skipCurrentTag(parser);
                        continue;
                    }
                }

                foundApp = true;
                if (!parseBaseApplication(pkg, res, parser, flags, outError)) {
                    return null;
                }
            }
            ...
        return pkg;
    }
```

parseBaseApkCommon 调用 parseBaseApplication 解析 application 节点。

```java
    private boolean parseBaseApplication(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError)
        throws XmlPullParserException, IOException {
        final ApplicationInfo ai = owner.applicationInfo;
        final String pkgName = owner.applicationInfo.packageName;
        ...
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > innerDepth)) {
            ...
            if (tagName.equals("activity")) {
                Activity a = parseActivity(owner, res, parser, flags, outError, cachedArgs, false,
                        owner.baseHardwareAccelerated);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                hasActivityOrder |= (a.order != 0);
                owner.activities.add(a);

            }
            ...
        return true;
    }

```

可以看出 parseBaseApplication 发现 activity 标签时，会使用 parseActivity 解析 activity。

parseActivity 方法如下：

```java
    private Activity parseActivity(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError, CachedComponentArgs cachedArgs,
            boolean receiver, boolean hardwareAccelerated)
            throws XmlPullParserException, IOException {
        ...
        while ((type=parser.next()) != XmlPullParser.END_DOCUMENT
               && (type != XmlPullParser.END_TAG
                       || parser.getDepth() > outerDepth)) {
            ...
            if (parser.getName().equals("intent-filter")) {
                ActivityIntentInfo intent = new ActivityIntentInfo(a);
                if (!parseIntent(res, parser, true /*allowGlobs*/, true /*allowAutoVerify*/,
                        intent, outError)) {
                    return null;
                }
                ...
            } else if (parser.getName().equals("meta-data")) {
                if ((a.metaData = parseMetaData(res, parser, a.metaData,
                        outError)) == null) {
                    return null;
                }
            ...
        return a;
    }

````

与 parseBaseApplication 类似，parseActivity 内部逻辑也是遍历子标签，并调用相应方法对其进行解析，比如 parseIntent 和 parseMetaData。