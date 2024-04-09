---
title: KubeVela 源码阅读——插件渲染
author: siegelion
date: 2022/6/29 15:15
tags: [KubeVela, 云原生]
categories: [笔记]
---



# Addon

> KubeVela本身是一个比较新的项目，正处于高速发展期，因此很多设计可能会在短时间内就迎来变化，因此本文介绍是`V1.4~V1.5`版本时KubeVela内部加载一个插件的流程。
> 


# 插件目录结构

[自定义插件 | KubeVela](https://kubevela.io/zh/docs/platform-engineers/addon/intro)

以上暂时是KubeVela V1.4版本的插件结构，在V1.5版本后会将插件使用的`VelaQL`实例view文件单独渲染，因此引入了一个新的文件目录views，下文都是以新的目录结构进行论述。新的插件目录结构如下：

```bash
├── resources/
├── definitions/
├── schemas/
├── views/
├── README.md
├── metadata.yaml
└── template.yaml
```

# 插件渲染流程

## 加载安装包

*`EnableAddon`*

> pkg/addon/helper.go
> 

```go
func EnableAddon(ctx context.Context, name string, version string, cli client.Client, discoveryClient *discovery.DiscoveryClient, apply apply.Applicator, config *rest.Config, r Registry, args map[string]interface{}, cache *Cache) error {
	h := NewAddonInstaller(ctx, cli, discoveryClient, apply, config, &r, args, cache)
	pkg, err := h.loadInstallPackage(name, version)
	if err != nil {
		return err
	}
	err = h.enableAddon(pkg)
	if err != nil {
		return err
	}
	return nil
}
```

这里比较重要的是两个函数：`loadInstallPackage` 和 `enableAddon`

`loadInstallPackage` 

> pkg/addon/addon.go
> 

```go
func (h *Installer) loadInstallPackage(name, version string) (*InstallPackage, error) {
	var installPackage *InstallPackage
	var err error
	if !IsVersionRegistry(*h.r) {
		...
		var uiData *UIData
		uiData, err = h.cache.GetUIData(*h.r, name, version)
		...
		// enable this addon if it's invisible
		installPackage, err = h.r.GetInstallPackage(&meta, uiData)
		...
	} else {
		versionedRegistry := BuildVersionedRegistry(h.r.Name, h.r.Helm.URL, &common.HTTPOption{
			Username: h.r.Helm.Username,
			Password: h.r.Helm.Password,
		})
		installPackage, err = versionedRegistry.GetAddonInstallPackage(context.Background(), name, version)
		...
	}

	return installPackage, nil
}
```

这个函数首先检查安装插件的仓库是否是一个支持多版本的仓库，这个检查也很简单直接判断是否设置了`helm`字段。

若不支持多版本（本地或者OSS），则直接安装插件，否则构建一个多版本的仓库对象，然后根据需要的addon的`name`和`version`安装插件。

## 加载插件安装包

### 不支持多版本安装包

`GetInstallPackage`

> pkg/addon/source.go
> 

```go
// GetInstallPackage get install package which is all needed to enable an addon from addon registry
func (r *Registry) GetInstallPackage(meta *SourceMeta, uiData *UIData) (*InstallPackage, error) {
	...
	return GetInstallPackageFromReader(reader, meta, uiData)
}
```

*`GetInstallPackageFromReader`*

> pkg/addon/addon.go
> 

```go
// GetInstallPackageFromReader get install package of addon from Reader, this is used to enable an addon
func GetInstallPackageFromReader(r AsyncReader, meta *SourceMeta, uiData *UIData) (*InstallPackage, error) {
	addonContentsReader := map[string]func(a *InstallPackage, reader AsyncReader, readPath string) error{
		TemplateFileName: readTemplate,
		ResourcesDirName: readResFile,
		DefSchemaName:    readDefSchemaFile,
		ViewDirName:      readViewFile,
	}
	ptItems := ClassifyItemByPattern(meta, r)

	// Read the installed data from UI metadata object to reduce network payload
	var addon = &InstallPackage{
		Meta:           uiData.Meta,
		Definitions:    uiData.Definitions,
		CUEDefinitions: uiData.CUEDefinitions,
		Parameters:     uiData.Parameters,
	}

	for contentType, method := range addonContentsReader {
		items := ptItems[contentType]
		for _, it := range items {
			err := method(addon, r, r.RelativePath(it))
			...
		}
	}

	return addon, nil
}
```

该函数将template、resource、schema、view分别加载进相应的字段中，用于下面启动插件时加载用。

### 多版本安装包

*`GetAddonInstallPackage`*

> pkg/addon/versioned_registry.go
> 

```go
func (i *versionedRegistry) GetAddonInstallPackage(ctx context.Context, addonName, version string) (*InstallPackage, error) {
	// 加载插件，重要
	wholePackage, err := i.loadAddon(ctx, addonName, version)
	if err != nil {
		return nil, err
	}
	return &wholePackage.InstallPackage, nil
}
```

`loadAddon`

> pkg/addon/versioned_registry.go
> 

```go
func (i versionedRegistry) loadAddon(ctx context.Context, name, version string) (*WholeAddonPackage, error) {
	versions, err := i.h.ListVersions(i.url, name, false, i.Opts)
	if err != nil {
		return nil, err
	}
	if len(versions) == 0 {
		return nil, ErrNotExist
	}
	sort.Sort(sort.Reverse(versions))
	addonVersion, availableVersions := chooseVersion(version, versions)
	if addonVersion == nil {
		return nil, fmt.Errorf("specified version %s not exist", version)
	}
	for _, chartURL := range addonVersion.URLs {
		if !utils.IsValidURL(chartURL) {
			chartURL, err = utils.JoinURL(i.url, chartURL)
			if err != nil {
				return nil, fmt.Errorf("cannot join versionedRegistryURL %s and chartURL %s, %w", i.url, chartURL, err)
			}
		}
		archive, err := common.HTTPGetWithOption(ctx, chartURL, i.Opts)
		if err != nil {
			continue
		}
		bufferedFile, err := loader.LoadArchiveFiles(bytes.NewReader(archive))
		if err != nil {
			continue
		}
		addonPkg, err := loadAddonPackage(name, bufferedFile)
		if err != nil {
			return nil, err
		}
		addonPkg.AvailableVersions = availableVersions
		addonPkg.RegistryName = i.name
		return addonPkg, nil
	}
	return nil, fmt.Errorf("cannot fetch addon package")
}
```

`loadAddon`是一个很重要的代码，被反复使用。

1. 主要的逻辑为首先使用*`ListVersions`*函数加载插件的多个版本
   
    *`ListVersions`*
    
    ```go
    func (h *Helper) ListVersions(repoURL string, chartName string, skipCache bool, opts *common.HTTPOption) (repo.ChartVersions, error) {
    	i, err := h.GetIndexInfo(repoURL, skipCache, opts)
    	if err != nil {
    		return nil, err
    	}
    	return i.Entries[chartName], nil
    }
    ```
    
    该函数首先获取chart repo的`index.yaml`文件，该文件对象的类型为`*repo.IndexFile`该类型的结构为：
    
    ```go
    type IndexFile struct {
        ServerInfo  map[string]interface{}   `json:"serverInfo,omitempty"`
        APIVersion  string                   `json:"apiVersion"`
        Generated   time.Time                `json:"generated"`
        Entries     map[string]ChartVersions `json:"entries"`
        PublicKeys  []string                 `json:"publicKeys,omitempty"`
        Annotations map[string]string        `json:"annotations,omitempty"`
    }
    ```
    
    该对象的`Entries`字段为一个map对象，该map的<key,value>对为addon名和该addon的多个版本。
    
2. 然后对多个版本进行由大到小的排序，然后根据`version`信息确定要获取的对应版本的addon。
3. 根据该版本addon的url，获取对应的压缩包文件，然后解压缩后将`meta.yaml`对应的字节码加载进`*WholeAddonPackage`类型的addonPkg对象中。
4. `loadAddonPackage` 从压缩包的schemas、resources、views目录中加载相关组件信息。
   
    `loadAddonPackage` 
    
    ```go
    func loadAddonPackage(addonName string, files []*loader.BufferedFile) (*WholeAddonPackage, error) {
    	mr := MemoryReader{Name: addonName, Files: files}
    	metas, err := mr.ListAddonMeta()
    	...
    	meta := metas[addonName]
    	addonUIData, err := GetUIDataFromReader(&mr, &meta, UIMetaOptions)
    	...
    	installPackage, err := GetInstallPackageFromReader(&mr, &meta, addonUIData)
    	...
    }
    ```
    
    `GetInstallPackageFromReader`
    
    > pkg/addon/addon.go
    > 
    
    ```go
    // GetInstallPackageFromReader get install package of addon from Reader, this is used to enable an addon
    func GetInstallPackageFromReader(r AsyncReader, meta *SourceMeta, uiData *UIData) (*InstallPackage, error) {
    	addonContentsReader := map[string]func(a *InstallPackage, reader AsyncReader, readPath string) error{
    		TemplateFileName: readTemplate,
    		ResourcesDirName: readResFile,
    		DefSchemaName:    readDefSchemaFile,
    		ViewDirName:      readViewFile,
    	}
    	ptItems := ClassifyItemByPattern(meta, r)
    
    	// Read the installed data from UI metadata object to reduce network payload
    	var addon = &InstallPackage{
    		Meta:           uiData.Meta,
    		Definitions:    uiData.Definitions,
    		CUEDefinitions: uiData.CUEDefinitions,
    		Parameters:     uiData.Parameters,
    	}
    
    	for contentType, method := range addonContentsReader {
    		items := ptItems[contentType]
    		for _, it := range items {
    			err := method(addon, r, r.RelativePath(it))
    			...
    		}
    	}
    
    	return addon, nil
    }
    ```
    
    该函数将template、resource、schema、view分别加载进相应的字段中，用于下面启动插件时加载用。
    
5. addonPkg对象的`AvailableVersions`字段中还存储了该addon中全部可用的版本信息。

## 启动插件

`enableAddon`

> pkg/addon/addon.go
> 

```go
func (h *Installer) enableAddon(addon *InstallPackage) error {
	var err error
	h.addon = addon

	if !h.skipVersionValidate {
		err = checkAddonVersionMeetRequired(h.ctx, addon.SystemRequirements, h.cli, h.dc)
		if err != nil {
			version := h.getAddonVersionMeetSystemRequirement(addon.Name)
			return VersionUnMatchError{addonName: addon.Name, err: err, userSelectedAddonVersion: addon.Version, availableVersion: version}
		}
	}

	if err = h.installDependency(addon); err != nil {
		return err
	}
	if err = h.dispatchAddonResource(addon); err != nil {
		return err
	}
	// we shouldn't put continue func into dispatchAddonResource, because the re-apply app maybe already update app and
	// the suspend will set with false automatically
	if err := h.continueOrRestartWorkflow(); err != nil {
		return err
	}
	return nil
}
```

启动插件的流程主要有以下几点：

1. 检查插件的版本是否满足版本要求
2. 检查插件是否依赖其他组件，如果有这些组件是否已经安装
3. 解压渲染相应的资源进行渲染
4. 继续前面的工作流

### 检查版本需求

`checkAddonVersionMeetRequired`

> pkg/addon/addon.go
> 

```go
// checkAddonVersionMeetRequired will check the version of cli/ux and kubevela-core-controller whether meet the addon requirement, if not will return an error
// please notice that this func is for check production environment which vela cli/ux or vela core is officalVersion
// if version is for test or debug eg: latest/commit-id/branch-name this func will return nil error
func checkAddonVersionMeetRequired(ctx context.Context, require *SystemRequirements, k8sClient client.Client, dc *discovery.DiscoveryClient) error {
	...

	// if not semver version, bypass check cli/ux. eg: {branch name/git commit id/UNKNOWN}
	if version2.IsOfficialKubeVelaVersion(version2.VelaVersion) {
		res, err := checkSemVer(version2.VelaVersion, require.VelaVersion)
	}

	// check vela core controller version
	imageVersion, err := fetchVelaCoreImageTag(ctx, k8sClient)

	// if not semver version, bypass check vela-core.
	if version2.IsOfficialKubeVelaVersion(imageVersion) {
		res, err := checkSemVer(imageVersion, require.VelaVersion)
	}

	// discovery client is nil so bypass check kubernetes version
	if dc == nil {
		return nil
	}

	k8sVersion, err := dc.ServerVersion()
		
	// if not semver version, bypass check kubernetes version.
	if version2.IsOfficialKubeVelaVersion(k8sVersion.GitVersion) {
		res, err := checkSemVer(k8sVersion.GitVersion, require.KubernetesVersion)
	}

	return nil
}
```

对插件进行系统版本要求的检查时，主要有两大种类型的版本要求：

- Vela 版本
    - Vela UX 版本
    - Vela Core 版本
- Kubernetes 版本

### 检查其他插件依赖

`installDependency`

> pkg/addon/addon.go
> 

```go
func (h *Installer) installDependency(addon *InstallPackage) error {
	var app v1beta1.Application
	for _, dep := range addon.Dependencies {
		err := h.cli.Get(h.ctx, client.ObjectKey{
			Namespace: types.DefaultKubeVelaNS,
			Name:      Convert2AppName(dep.Name),
		}, &app)
		if err == nil {
			continue
		}
		if !apierrors.IsNotFound(err) {
			return err
		}
		// always install addon's latest version
		depAddon, err := h.loadInstallPackage(dep.Name, "")
		if err != nil {
			return err
		}
		depHandler := *h
		depHandler.args = nil
		if err = depHandler.enableAddon(depAddon); err != nil {
			return errors.Wrap(err, "fail to dispatch dependent addon resource")
		}
	}
	return nil
}
```

### 部署插件

`dispatchAddonResource`

> pkg/addon/addon.go
> 

```go
func (h *Installer) dispatchAddonResource(addon *InstallPackage) error {
	app, err := RenderApp(h.ctx, addon, h.cli, h.args)
	...
	defs, err := RenderDefinitions(addon, h.config)
	...
	schemas, err := RenderDefinitionSchema(addon)
	...
	views, err := RenderViews(addon)
	...

	for _, def := range defs {
		addOwner(def, app)
		err = h.apply.Apply(h.ctx, def, apply.DisableUpdateAnnotation())
		if err != nil {
			return err
		}
	}

	for _, schema := range schemas {
		addOwner(schema, app)
		err = h.apply.Apply(h.ctx, schema, apply.DisableUpdateAnnotation())
		if err != nil {
			return err
		}
	}

	for _, view := range views {
		addOwner(view, app)
		err = h.apply.Apply(h.ctx, view, apply.DisableUpdateAnnotation())
		if err != nil {
			return err
		}
	}

	...
	return nil
}
```

这里就是将加载进、存储在对应字段中的definition、schema、view渲染成k8s对象，然后应用到集群中，至此一个插件的安装流程就结束了。