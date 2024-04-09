---
title: KubeVela 源码阅读——控制器处理
author: siegelion
date: 2022/7/15 15:15
tags: [KubeVela, 云原生]
categories: [笔记]
---

KubeVela 以应用为中心，因此对控制器处理逻辑的分析中选取 Application 控制器作为主要的分析对象。

# 控制器启动

Application 控制器是 OAM 控制集中的一个控制器，在控制器启动的流程中，通过调用了 OAM 控制集的启动，间接启动了该控制器。

```go
// Setup adds a controller that reconciles AppRollout.
func Setup(mgr ctrl.Manager, args core.Args) error {
	reconciler := Reconciler{
		Client:   mgr.GetClient(),
		Scheme:   mgr.GetScheme(),
		Recorder: event.NewAPIRecorder(mgr.GetEventRecorderFor("Application")),
		dm:       args.DiscoveryMapper,
		pd:       args.PackageDiscover,
		options:  parseOptions(args),
	}
	return reconciler.SetupWithManager(mgr)
}
```

启动控制的最后一步，将初始化的控制器安装至控制器管理器。

# 安装至管理器

将控制器安装至管理器的流程如下图所示：

![image-20220716195614713](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20220716195614713.png)

```go
// SetupWithManager install to manager
func (r *Reconciler) SetupWithManager(mgr ctrl.Manager) error {
	// If Application Own these two child objects, AC status change will notify application controller and recursively update AC again, and trigger application event again...
	return ctrl.NewControllerManagedBy(mgr).
		Watches(&source.Kind{
			Type: &v1beta1.ResourceTracker{},
		}, ctrlHandler.Funcs{
			...
		}).
		WithOptions(controller.Options{
			...
		}).
		WithEventFilter(predicate.Funcs{
            ...
		}).
		For(&v1beta1.Application{}).
		Complete(r)
}
```

`Watch`函数是一个较低层次的，用于被监控资源类型的函数，`For` 函数与 `Own` 函数的功能也可以使用 `Watch` 函数实现。但不同的是，`Watch` 函数可以自行提供 `Handler` 函数。此处`Watch`函数关注的`ResourceTracker`类型，用于实现垃圾回收。

`For` 函数的对象是 reconcile 的对象，该类型的对象的 create、delete、update 事件会触发对应的`Reconcile`函数，一个 controller builder 中不能不存在 `For` 函数。

# Reconcile

`Reconcile`函数会在控制器对应类型对象的状态与声明状态不符时自动被调用。下图是`Reconcile`的流程示意图。

![Reconcile](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20220713120400517.png)

## appfile

###  生成 parser

生成parser的过程比较简单，较为关键的是parser实例化的过程中`tmplLoader`装载了一个`LoadTemplate`函数，这个函数作为一个回调函数会在后续的流程中调用，用于解析component、workflowStep等的定义。

该函数会根据传入capability definition的类型，获取对应的definition，然后生成对应的template。

---

生成appfile有两种途径，一种是从appRevision生成，另一种是从app生成，appRevision可以看作是app的一种快照。生成appfile前首先需要判断一下，当前的appRevision的版本是否与发布app的版本对应，若是则选择从appResion生成，否则则选择从app生成。

为何有了appRevision这一中间产物，为什么不每次都从app生成，这个答案可以从两者结构体的不同中找到答案。

- app.Spec

    ```go
    type ApplicationSpec struct {
        Components []common.ApplicationComponent 
        Policies   []AppPolicy                   
        Workflow   *Workflow                    
    }
    ```

- appRevision.Spec

    ```go
    type ApplicationRevisionSpec struct {
        Application             Application                 
        ComponentDefinitions    map[string]ComponentDefinition     
        WorkloadDefinitions     map[string]WorkloadDefinition    
        TraitDefinitions        map[string]TraitDefinition         
        ScopeDefinitions        map[string]ScopeDefinition         
        PolicyDefinitions       map[string]PolicyDefinition        
        WorkflowStepDefinitions map[string]WorkflowStepDefinition  
        ScopeGVK                map[string]v1.GroupVersionKind
        Policies                map[string]v1alpha1.Policy         
        Workflow                *v1alpha1.Workflow                 
        ReferredObjects         []common.ReferredObject           
    }
    ```

appRevision.Spec相当于是对app.Spec进行了一个拆解转化，这个过程需要一定的运算，后续步骤中进行的运算也即是进行这个过程，这个运算的消耗可以通过快照的方法进行节省，所以在判断快照的版本合法的情况下直接从快照生成appfile。

### 从 app 生成 appfile

我们可以通过官网提供的一个定义 app 的 yaml 文件与源码中 Application 结构体相对应来了解 app 的数据结构。 

![app 结构](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20220715115300290.png)

 此处我们以从app生成为例，分析appfile的生成流程。

![appfile 生成流程](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20220715113128207.png)

#### workload

其中比较重要的步骤为`parseWorkload`，在装载步骤步骤中的三种Definition也是来自解析出的workload。

`parseWorkload`的作用可以从函数的注释中获取到，其作用是”解析一个 ApplicationComponent 并生成一个包含 Appfile 所需的所有信息的 Workload ”。

> parseWorkload resolve an ApplicationComponent and generate a Workload containing ALL information required by an Appfile.

1. 生成workload的过程中，首先需要利用我们上文 parser 提到的 `LoadTemplate`函数，此处使用该函数是传入的capability definition 为 componentDefinition，因此是生成 component 对应的 template。

2. 然后使用 template 解析 component 中的参数将其转化为 workload，也即`component.properties`部分。参数详细信息参照：https://kubevela.io/zh/docs/next/end-user/components/references。

    ![component.properties](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20220715122650234.png)
    
3. 加载部分参数，包括加载 trait，加载scope。

#### workflowStep

`parseWorkflowSteps`是另一个较为重要的步骤，关于工作流的详细信息可以参照官网文档：https://kubevela.io/zh/docs/next/end-user/workflow/overview。

![workflowStep](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20220715180203860.png)

- `loadWorkflowToAppfile`的设计非常巧妙，使用了函数链式调用的设计，将多个步骤进行了串联：

    ![workflowStep](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20220715183409001.png)

    - 其中`DeployWorkflowStepGenerator`会将 policy，转换为`deploy`类型的 step，这里进行转换的 policy  的类型为 override 与 topology 两种类型。

    - `ApplyComponentWorkflowStepGenerator`则将会将`app.component`转换为`apply-component`类型的 step。

- `parseWorkflowStep`则遍历上文生成的 step，获取对应类型的definition，将其装载至 appfile 的`RelatedWorkflowStepDefinitions`字段中。

## workflow step

### 生成 taskRunner

这部分对应的函数是`GenerateApplicationSteps`及相关函数，这里所做的处理是将上文 的step （`v1beta1.WorkflowStep`），转换为一种新类型的 step（`wfTypes.TaskRunner`）。

<img src="https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20220716173000475.png" alt="image-20220716173000475" style="zoom:50%;" />

在生成 taskRunner 之前，首先要注册数个Provider，这些 Provider 会提供相应的能力，后续会使用他们的能力去实现相应的操作。

在 taskDiscover 的过程中，实际上在其内部还封装了一个 TaskLoader，这二者的作用都是在下面的流程中加载tempplate。

GetTaskGenerator 的过程中，会使用分别使用上文提到的taskDiscover 与 TaskLoader 加载 CUE 类型的 template，并作为参数传入 makeTaskGenerator，makeTaskGenerator 的逻辑比较复杂，这里它并没有直接运行任何函数，而是返回了一个闭包函数，这个函数被赋值给 genTask 变量，接下来会执行 genTask，进而调用这个闭包函数。

这个闭包函数中主要创建了 taksRunner 类型的变量 tRunner，并初始化了他的三个字段，其中我们重点关注`run` 这个字段，该字段为一个函数类型的字段，这里会将一个匿名函数赋值给他，tRuner 作为返回值进行返回，至此便将 step 成功转换为了 taskRunner。

### 执行 task

该过程对应`ExecuteSteps`所进行的操作。执行该函数时会初始化一个 workflow 的 engine，然后调用 engine 的 Run 方法。Run 方法会运行上文生成的 taskRunner ，执行方式有两种：StepByStep 顺序执行以及 DAG 并行执行，关于 workflow 的执行方式的详细信息可以参考：https://kubevela.io/zh/docs/next/end-user/workflow/overview。

这两个执行方式最后都会遍历 taskRunner，然后调用其 Run 方法，该方法实际上也即上文我们在创建 taksRunner 时，设置在其 run 字段上的函数。

现在我们可以回头重新来分析一下，这个 run 字段上的函数做了怎样的操作：

![image-20220716193644472](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20220716193644472.png)

实际上该函数的操作分为两大部分：渲染CUE value与解析CUE value执行相应操作。

渲染CUE value前会将上文我们获取到的workflowStep definition的与workflow.Properties（参数）组合在一起，这两个部分组合在一起模板+参数就构成一个确定的workflowStep，之后将其加载为CUE value。

解析参数并执行相应操作的过程，主要由`doStep`函数完成，该函数的主要思路即为根据CUE value中的相关定义，获取相应的Provider中的相应能力（函数），执行相应的能力（函数）。