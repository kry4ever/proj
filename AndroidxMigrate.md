
#前言

之前在给抖音制定androidx迁移方案的时候写过一篇doc，里面详细写了迁移时遇到的问题以及解决方案。如果有需要迁移的同学可以看看文档。对androidx还不了解的同学，如何迁移？有什么环境要求？以及迁移有什么好处可以上官网看看。文章里写了如何使用我们自定义的迁移工具，今天我们就这个工具来讲讲遇到的挑战和源码实现吧

#官方工具真的好用吗？

google的确给出一个一键迁移的方案，即点击Migrate To Androidx，android studio会把源码里面用support的点都替换成对应androidx的，包括java、kt、xml、gradle依赖等，但是真的好用吗？
可以看到有些工程打开直接是置灰的，可能跟工程结构有关系。。oh, migrate failed..

![avatar](https://note.youdao.com/yws/public/resource/90e0e85a89029ebff0ad090ff0603f41/xmlnote/DF41F83744414E9AAC229BAA438E06F0/7604)

或者也有人说

![avatar](https://note.youdao.com/yws/public/resource/90e0e85a89029ebff0ad090ff0603f41/xmlnote/93DF04859EEE4911AC9473135397AE62/7602)

像类似问题还有一些，比如R这个类的替换问题等，其实还有一个大问题，官方这个点击后会让你保存当前工程到一个zip文件作为backup，以免出什么大问题。可是对于抖音这种快速发车的大型工程而言，怎么能在一个时间点停下来去backup一个zip呢？而且zip backup好了也没法替换到源工程的。
对于这种多module的工程我们需要一个替换脚本，能多次的、稳定的、能回溯的脚本替换方案，考虑到每个子仓库也可能需要迁移，因此我采用了gradle脚本而不是python脚本，下面看看怎么做的吧？

code就不贴了

然后只要用了这个gradle插件，就可以一键执行任务了。
./gradlew fullMigrate

或者如果出问题想一键回退
./gradlew fullMigrate -Preverse=true

如此一来我们可以快速的，多次的尝试替换，如果有不符合预期的地方用git reset代码，或者用我们的reverse功能就可以了
现在主仓替换完了，子仓库呢？
google是要求我们一刀切的，即要么全是androidx，要么全是support，但是此时如果我们有子仓库无法迁移到androidx怎么办？比如直播，工具线，他们都是给其他app使用的，如果切换到androidx必然导致其他使用support的宿主无法编译。
先说一个前置的问题？为什么androidx不能与support兼容呢？首先这样会导致混乱，代码一样就是package name不同就存在两份代码，这不合理。那依赖的那些aar怎么办？它们好些都是三方库已经编译好了，它里面用support怎么处理？不用担心，因为官方在agp里面用的jetifier工具就是处理aar的，它会做字节码替换，把support的字节码替换成androidx的。可是源码编译怎么办呢？子仓的接口要求使用support的activity，但是主仓只有androidx的activity，必然导致编译错误，因此我们想能否hook子仓的产物呢？让它变成androidx的接口不就可以了吗？OK，先上架构图



![avatar](https://note.youdao.com/yws/public/resource/90e0e85a89029ebff0ad090ff0603f41/xmlnote/5844BA784F9E4DDD84C6845D6CC7783B/7603)



具体分析可以看文档，这里简单说下就是去hook编译后的jar文件和xml，让它们变成androidx的接口，先强调下我们解决的是下层模块是support接口，上层模块是androidx的不兼容问题
我们会在gradle task执行前后去hook。下面是伪代码

project.gradle.taskGraph.addTaskExecutionListener(new TaskExecutionListener() {
    @Override
    void beforeExecute(Task task) {
        try {
            UtilsKt.hookClasspathBeforeCompile(task)
            UtilsKt.hookClasspathBeforeKapt(task)
        } catch (Exception e) {
            e.printStackTrace()
        }
    }

    @Override
    void afterExecute(Task task, TaskState taskState) {
        try {
            UtilsKt.generateJetifierClassIfNeed(enableHook, task)
            UtilsKt.generateJetifierXmlIfNeed(enableHook, xmlReplacement, task)
        } catch (Exception e) {
            e.printStackTrace()
        }
    }
})

在task执行after之后，我们需要hook原来这个task的产物，因为它原来只有support的接口，我们需要用jetifier工具生产一份androidx的jar才能让上层顺利编译，也就是generateJetifierClassIfNeed做的工作

fun generateJetifierClassIfNeed(enableHook: Boolean, task: Task) {
    val jetifyProcessor: Processor = Processor.createProcessor(
            ConfigParser.loadDefaultConfig()!!
        )
        task.outputs.files.forEach { file ->
            if (!file.isDirectory) {
                if (!file.name.toLowerCase().endsWith(".jar")) {
                        return@forEach
                    }
                    if (jetifyProcessor.isNewDependencyFile(file)) {
                        return@forEach
                    }
                    if (jetifyProcessor.isOldDependencyFile(file)) {
                        return@forEach
                    }
                    val fileSet = setOf(FileMapping(file, findJetifierFile(file)))
                    val transformedFile = jetifyProcessor.transform(fileSet, false).single()
                    if (transformedFile.absolutePath != file.absolutePath) {
                        file.renameTo(findBackupFile(file))
                        findJetifierFile(file).renameTo(file)
                    }
            }
        }
}


xml的替换就不多说了，因为xml只是文本替换，跟上面说的替换工具是一致的

#总结

其实整个模型就是生产者和消费者的模型，afterTask生成的jar的任务就是生产者的角色，需要参与编译的task，javac、kotlinc等就是消费者，它们需要选择合适产物加入自己的编译路径参与编译。经过这个gradle插件，我们解决了脚本一键迁移和回滚问题，解决了support的子仓库和androidx的主仓库的源码依赖编译问题，并实现了渐进式迁移，可以多期的，从上置下的迁移到androidx。
