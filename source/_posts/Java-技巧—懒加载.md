---
title: Java 技巧—懒加载
date: 2017-10-28 16:56:33
tags:
    - Java
categories: 工具
---

```Java
package io.github.binglau;

import java.util.function.Supplier;

/**
 * 文件描述: 懒加载大对象 Heavy
 * 利用类加载机制达到 LazyLoad 为单例
 */

public final class LazyLoad {
    private LazyLoad() {
    }

    private static class Holder {
        private static final LazyLoad INSTANCE = new LazyLoad();
    }

    public static final LazyLoad getInstance() {
        return Holder.INSTANCE;
    }

    private Supplier<Heavy> heavy = this::createAndCacheHeavy;
    private Normal normal = new Normal();

    public Heavy getHeavy() {
        return heavy.get();
    }

    public Normal getNormal() {
        return normal;
    }

    private synchronized Heavy createAndCacheHeavy() {
        class HeavyFactory implements Supplier<Heavy> {
            private final Heavy heavyInstance = new Heavy();
            @Override
            public Heavy get() {
                return heavyInstance;
            }
        }

        if (!HeavyFactory.class.isInstance(heavy)) {
            heavy = new HeavyFactory();
        }

        return heavy.get();
    }

    public static void main(String[] args) throws InterruptedException {
        LazyLoad.getInstance().getNormal();
        LazyLoad.getInstance().getHeavy();
    }

}

class Heavy {
    public Heavy() {
        System.out.println("Creating Heavy ...");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("... Heavy created");
    }
}

class Normal {
    public Normal() {
        System.out.println("Creating Normal ...");
        System.out.println("... Normal created");
    }
}
```
