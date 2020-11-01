---
title: gradle版本冲突如何处理
tags: [gradle]
categories: [gradle]
date: 2020-11-01 16:30:05
description: 前言、版本冲突处理、gradle最新版本判断代码、结论
---

# 前言

当我们遇到gradle的版本依赖冲突时，一般可以通过force语句来处理：
```
resolutionStrategy {
	force 'xxxxxx'
}
```

当如果我们不使用force语句时，原本的gradle版本冲突时如何处理的呢？

# 版本冲突处理

当我们添加了gradle的源码依赖后：
```
implementation gradleApi()
implementation localGroovy()
```

我们可以找到下面的枚举类：
```
/**
 * The conflict resolution
 */
public enum ConflictResolution {
    strict,
    latest,
    preferProjectModules
}
```

其中：
- strict：严格模式，通过 failOnVersionConflict() 来设置，表示如果存在依赖冲突则失败
- latest：选择最新的，默认选项
- preferProjectModules：没用过，官方说明是 prefer modules that are part of this build (multi-project or composite build) over external modules

一般而言，我们都是默认选择最新的版本

但由于依赖库的版本号是字符串，而非数字，那么gradle默认的实现是如何判断哪个版本是最新呢？

# gradle最新版本判断代码

枚举latest对应的版本处理类是 LatestModuleConflictResolver

他的构造函数中需要传入处理版本冲突的版本对比器，默认实现代码如下：
```
/**
 * This comparator considers `1.1.1 == 1-1-1`.
 * @see StaticVersionComparator
 */
public class DefaultVersionComparator implements VersionComparator {
    private final Comparator<Version> baseComparator = new StaticVersionComparator();

    public int compare(Versioned element1, Versioned element2) {
        Version version1 = element1.getVersion();
        Version version2 = element2.getVersion();
        return baseComparator.compare(version1, version2);
    }

    @Override
    public Comparator<Version> asVersionComparator() {
        return baseComparator;
    }
}
```

其中StaticVersionComparator代码如下：
```
/**
 * Allows for comparison of Version instances.
 * Note that this comparator only considers the 'parts' of a version, and does not consider the part 'separators'.
 * This means that it considers `1.1.1 == 1-1-1 == 1.1-1`, and should not be used in cases where this is important.
 * One example where this comparator is inappropriate is if versions should be retained in a TreeMap/TreeSet.
 */
class StaticVersionComparator implements Comparator<Version> {
    private static final Map<String, Integer> SPECIAL_MEANINGS =
            ImmutableMap.of("dev", -1, "rc", 1, "release", 2, "final", 3);

    /**
     * Compares 2 versions. Algorithm is inspired by PHP version_compare one.
     */
    public int compare(Version version1, Version version2) {
        if (version1.equals(version2)) {
            return 0;
        }

        String[] parts1 = version1.getParts();
        String[] parts2 = version2.getParts();
        Long[] numericParts1 = version1.getNumericParts();
        Long[] numericParts2 = version2.getNumericParts();

        int i = 0;
        for (; i < parts1.length && i < parts2.length; i++) {
            String part1 = parts1[i];
            String part2 = parts2[i];

            Long numericPart1 = numericParts1[i];
            Long numericPart2 = numericParts2[i];

            boolean is1Number = numericPart1 != null;
            boolean is2Number = numericPart2 != null;

            if (part1.equals(part2)) {
                continue;
            }
            if (is1Number && !is2Number) {
                return 1;
            }
            if (is2Number && !is1Number) {
                return -1;
            }
            if (is1Number && is2Number) {
                return numericPart1.compareTo(numericPart2);
            }
            // both are strings, we compare them taking into account special meaning
            Integer sm1 = SPECIAL_MEANINGS.get(part1.toLowerCase(Locale.US));
            Integer sm2 = SPECIAL_MEANINGS.get(part2.toLowerCase(Locale.US));
            if (sm1 != null) {
                sm2 = sm2 == null ? 0 : sm2;
                return sm1 - sm2;
            }
            if (sm2 != null) {
                return -sm2;
            }
            return part1.compareTo(part2);
        }
        if (i < parts1.length) {
            return numericParts1[i] == null ? -1 : 1;
        }
        if (i < parts2.length) {
            return numericParts2[i] == null ? 1 : -1;
        }

        return 0;
    }
}
```

其中使用到的Version类的解析数据代码如下：
```
public class VersionParser implements Transformer<Version, String> {
    public static final VersionParser INSTANCE = new VersionParser();

    public VersionParser() {
    }

    @Override
    public Version transform(String original) {
        List<String> parts = new ArrayList<String>();
        boolean digit = false;
        int startPart = 0;
        int pos = 0;
        int endBase = 0;
        int endBaseStr = 0;
        for (; pos < original.length(); pos++) {
            char ch = original.charAt(pos);
            if (ch == '.' || ch == '_' || ch == '-' || ch == '+') {
                parts.add(original.substring(startPart, pos));
                startPart = pos + 1;
                digit = false;
                if (ch != '.' && endBaseStr == 0) {
                    endBase = parts.size();
                    endBaseStr = pos;
                }
            } else if (ch >= '0' && ch <= '9') {
                if (!digit && pos > startPart) {
                    if (endBaseStr == 0) {
                        endBase = parts.size() + 1;
                        endBaseStr = pos;
                    }
                    parts.add(original.substring(startPart, pos));
                    startPart = pos;
                }
                digit = true;
            } else {
                if (digit) {
                    if (endBaseStr == 0) {
                        endBase = parts.size() + 1;
                        endBaseStr = pos;
                    }
                    parts.add(original.substring(startPart, pos));
                    startPart = pos;
                }
                digit = false;
            }
        }
        if (pos > startPart) {
            parts.add(original.substring(startPart, pos));
        }
        DefaultVersion base = null;
        if (endBaseStr > 0) {
            base = new DefaultVersion(original.substring(0, endBaseStr), parts.subList(0, endBase), null);
        }
        return new DefaultVersion(original, parts, base);
    }

    private static class DefaultVersion implements Version {
        private final String source;
        private final String[] parts;
        private final Long[] numericParts;
        private final DefaultVersion baseVersion;

        public DefaultVersion(String source, List<String> parts, DefaultVersion baseVersion) {
            this.source = source;
            this.parts = parts.toArray(new String[0]);
            this.numericParts = new Long[this.parts.length];
            for (int i = 0; i < parts.size(); i++) {
                this.numericParts[i] = Longs.tryParse(this.parts[i]);
            }
            this.baseVersion = baseVersion == null ? this : baseVersion;
        }

        @Override
        public String toString() {
            return source;
        }

        @Override
        public boolean equals(Object obj) {
            if (obj == this) {
                return true;
            }
            if (obj == null || obj.getClass() != getClass()) {
                return false;
            }
            DefaultVersion other = (DefaultVersion) obj;
            return source.equals(other.source);
        }

        @Override
        public int hashCode() {
            return source.hashCode();
        }

        @Override
        public boolean isQualified() {
            return baseVersion != this;
        }

        @Override
        public Version getBaseVersion() {
            return baseVersion;
        }

        public String[] getParts() {
            return parts;
        }

        @Override
        public Long[] getNumericParts() {
            return numericParts;
        }

        @Override
        public String getSource() {
            return source;
        }
    }
}
```

# 结论

由上面的代码可知：
- 我们表示版本的字符串，首先会根据 ._-+ 分隔符分为多个部分，每个部分中，只会存在数字或者非数字的字符串
- 对比版本新旧时，其实是对版本的每个部分进行对比
- 同一部分中，数字的部分版本大于非数字的字符串，如果同为数字，则比较大小
- 同为字符串的情况下，查看是否特殊字符串，dev表示最低级别版本，低于普通字符串，rc高于普通字符串，release更高，最高的级别是final
- 同为普通字符串的情况下，通过string的compare方法来进行对比
- 如果上面的对比情况均相等，则判断最后较长的版本中，最后一部分是否为数字，为数字则较长的版本号为最新