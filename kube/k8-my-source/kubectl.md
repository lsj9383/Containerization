# Kubectl 源码解析

[TOC]

## 概览

代码版本：[kubernetes v1.14.0](https://github.com/kubernetes/kubernetes/tree/v1.14.0)。

## 源码

### 入口

[cmd/kubectl/kubectl.go](https://github.com/kubernetes/kubernetes/blob/v1.14.0/cmd/kubectl/kubectl.go)

```go
func main() {
	rand.Seed(time.Now().UnixNano())

	// 构造 Command 对象
	command := cmd.NewDefaultKubectlCommand()

	// TODO: once we switch everything over to Cobra commands, we can go back to calling
	// cliflag.InitFlags() (by removing its pflag.Parse() call). For now, we have to set the
	// normalize func and add the go flag set by hand.
	pflag.CommandLine.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
	pflag.CommandLine.AddGoFlagSet(goflag.CommandLine)
	// cliflag.InitFlags()
	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		fmt.Fprintf(os.Stderr, "%v\n", err)
		os.Exit(1)
	}
}
```

### kubectl 根命令

```go
func NewDefaultKubectlCommand() *cobra.Command {
    // 构造带命令行参数的 Command 对象
	return NewDefaultKubectlCommandWithArgs(NewDefaultPluginHandler(plugin.ValidPluginFilenamePrefixes),
                                            os.Args,
                                            os.Stdin,
                                            os.Stdout,
                                            os.Stderr)

    // NewDefaultPluginHandler 用于 kubectl 扩展机制，该机制放文末分析
}

// creates the `kubectl` command with arguments
func NewDefaultKubectlCommandWithArgs(pluginHandler PluginHandler,
                                      args []string,
                                      in io.Reader,
                                      out, errout io.Writer) *cobra.Command {
    // 构造 Command 对象
	cmd := NewKubectlCommand(in, out, errout)

	if pluginHandler == nil {
		return cmd
	}

	if len(args) > 1 {
		cmdPathPieces := args[1:]

        // 如果指定的命令尚不存在，则仅查找合适的扩展可执行文件
        // 这是一种 kubectl 扩展机制
		if _, _, err := cmd.Find(cmdPathPieces); err != nil {
			if err := HandlePluginCommand(pluginHandler, cmdPathPieces); err != nil {
				fmt.Fprintf(errout, "Error: %v\n", err)
				os.Exit(1)
			}
		}
	}

	return cmd
}

func NewKubectlCommand(in io.Reader, out, err io.Writer) *cobra.Command {
	// 构造 Command 根命令
	cmds := &cobra.Command{
		Use:   "kubectl",
		Short: i18n.T("kubectl controls the Kubernetes cluster manager"),
		Long: templates.LongDesc(`
      kubectl controls the Kubernetes cluster manager.

      Find more information at:
            https://kubernetes.io/docs/reference/kubectl/overview/`),
		Run: runHelp,
		// Hook before and after Run initialize and write profiles to disk, respectively.
		PersistentPreRunE: func(*cobra.Command, []string) error { return initProfiling() },
		PersistentPostRunE: func(*cobra.Command, []string) error { return flushProfiling() },
		BashCompletionFunction: bashCompletionFunc,
	}

	// 全局命令函参数，可以用于子命令
	flags := cmds.PersistentFlags()
	flags.SetNormalizeFunc(cliflag.WarnWordSepNormalizeFunc) // Warn for "_" flags

	// Normalize all flags that are coming from other packages or pre-configurations
	// a.k.a. change all "_" to "-". e.g. glog package
	flags.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)

	addProfilingFlags(flags)

	// 创建 kubectl 使用的配置文件对象
	kubeConfigFlags := genericclioptions.NewConfigFlags(true).WithDeprecatedPasswordFlag()

	// 将 kubectl 使用的配置文件的字段关联到命令函参数上，以使得可以通过命令函参数覆盖
	kubeConfigFlags.AddFlags(flags)

	matchVersionKubeConfigFlags := cmdutil.NewMatchVersionFlags(kubeConfigFlags)
	matchVersionKubeConfigFlags.AddFlags(cmds.PersistentFlags())

	cmds.PersistentFlags().AddGoFlagSet(flag.CommandLine)

	f := cmdutil.NewFactory(matchVersionKubeConfigFlags)

	// Sending in 'nil' for the getLanguageFn() results in using
	// the LANG environment variable.
	//
	// TODO: Consider adding a flag or file preference for setting
	// the language, instead of just loading from the LANG env. variable.
	i18n.LoadTranslations("kubectl", nil)

	// From this point and forward we get warnings on flags that contain "_" separators
	cmds.SetGlobalNormalizationFunc(cliflag.WarnWordSepNormalizeFunc)

	ioStreams := genericclioptions.IOStreams{In: in, Out: out, ErrOut: err}

	groups := templates.CommandGroups{
		{
			Message: "Basic Commands (Beginner):",
			Commands: []*cobra.Command{
				create.NewCmdCreate(f, ioStreams),
				expose.NewCmdExposeService(f, ioStreams),
				run.NewCmdRun(f, ioStreams),
				set.NewCmdSet(f, ioStreams),
			},
		},
		{
			Message: "Basic Commands (Intermediate):",
			Commands: []*cobra.Command{
				explain.NewCmdExplain("kubectl", f, ioStreams),
				get.NewCmdGet("kubectl", f, ioStreams),
				edit.NewCmdEdit(f, ioStreams),
				delete.NewCmdDelete(f, ioStreams),
			},
		},
		{
			Message: "Deploy Commands:",
			Commands: []*cobra.Command{
				rollout.NewCmdRollout(f, ioStreams),
				rollingupdate.NewCmdRollingUpdate(f, ioStreams),
				scale.NewCmdScale(f, ioStreams),
				autoscale.NewCmdAutoscale(f, ioStreams),
			},
		},
		{
			Message: "Cluster Management Commands:",
			Commands: []*cobra.Command{
				certificates.NewCmdCertificate(f, ioStreams),
				clusterinfo.NewCmdClusterInfo(f, ioStreams),
				top.NewCmdTop(f, ioStreams),
				drain.NewCmdCordon(f, ioStreams),
				drain.NewCmdUncordon(f, ioStreams),
				drain.NewCmdDrain(f, ioStreams),
				taint.NewCmdTaint(f, ioStreams),
			},
		},
		{
			Message: "Troubleshooting and Debugging Commands:",
			Commands: []*cobra.Command{
				describe.NewCmdDescribe("kubectl", f, ioStreams),
				logs.NewCmdLogs(f, ioStreams),
				attach.NewCmdAttach(f, ioStreams),
				cmdexec.NewCmdExec(f, ioStreams),
				portforward.NewCmdPortForward(f, ioStreams),
				proxy.NewCmdProxy(f, ioStreams),
				cp.NewCmdCp(f, ioStreams),
				auth.NewCmdAuth(f, ioStreams),
			},
		},
		{
			Message: "Advanced Commands:",
			Commands: []*cobra.Command{
				diff.NewCmdDiff(f, ioStreams),
				apply.NewCmdApply("kubectl", f, ioStreams),
				patch.NewCmdPatch(f, ioStreams),
				replace.NewCmdReplace(f, ioStreams),
				wait.NewCmdWait(f, ioStreams),
				convert.NewCmdConvert(f, ioStreams),
				kustomize.NewCmdKustomize(ioStreams),
			},
		},
		{
			Message: "Settings Commands:",
			Commands: []*cobra.Command{
				label.NewCmdLabel(f, ioStreams),
				annotate.NewCmdAnnotate("kubectl", f, ioStreams),
				completion.NewCmdCompletion(ioStreams.Out, ""),
			},
		},
	}
	groups.Add(cmds)

	filters := []string{"options"}

	// Hide the "alpha" subcommand if there are no alpha commands in this build.
	alpha := NewCmdAlpha(f, ioStreams)
	if !alpha.HasSubCommands() {
		filters = append(filters, alpha.Name())
	}

	templates.ActsAsRootCommand(cmds, filters, groups...)

	for name, completion := range bashCompletionFlags {
		if cmds.Flag(name) != nil {
			if cmds.Flag(name).Annotations == nil {
				cmds.Flag(name).Annotations = map[string][]string{}
			}
			cmds.Flag(name).Annotations[cobra.BashCompCustom] = append(
				cmds.Flag(name).Annotations[cobra.BashCompCustom],
				completion,
			)
		}
	}

	cmds.AddCommand(alpha)
	cmds.AddCommand(cmdconfig.NewCmdConfig(f, clientcmd.NewDefaultPathOptions(), ioStreams))
	cmds.AddCommand(plugin.NewCmdPlugin(f, ioStreams))
	cmds.AddCommand(version.NewCmdVersion(f, ioStreams))
	cmds.AddCommand(apiresources.NewCmdAPIVersions(f, ioStreams))
	cmds.AddCommand(apiresources.NewCmdAPIResources(f, ioStreams))
	cmds.AddCommand(options.NewCmdOptions(ioStreams.Out))

	return cmds
}
```

### kubectl create 子命令实现

```go
func NewCmdCreate(f cmdutil.Factory, ioStreams genericclioptions.IOStreams) *cobra.Command {
    // create 子命令构造的上下文，缓存命令行参数数据
	o := NewCreateOptions(ioStreams)

	cmd := &cobra.Command{
		Use:                   "create -f FILENAME",
		DisableFlagsInUseLine: true,
		Short:                 i18n.T("Create a resource from a file or from stdin"),
		Long:                  createLong,
		Example:               createExample,
		Run: func(cmd *cobra.Command, args []string) {
            // 校验 create 文件名
			if cmdutil.IsFilenameSliceEmpty(o.FilenameOptions.Filenames, o.FilenameOptions.Kustomize) {
				ioStreams.ErrOut.Write([]byte("Error: must specify one of -f and -k\n\n"))
				defaultRunFunc := cmdutil.DefaultSubCommandRun(ioStreams.ErrOut)
				defaultRunFunc(cmd, args)
				return
			}

            // 补全必要参数
			cmdutil.CheckErr(o.Complete(f, cmd))

            // 校验命令函参数
			cmdutil.CheckErr(o.ValidateArgs(cmd, args))

            // 执行实际的核心逻辑
			cmdutil.CheckErr(o.RunCreate(f, cmd))
		},
	}

	// bind flag structs
	o.RecordFlags.AddFlags(cmd)

    // 绑定命令参数：-f / -k / -R
	cmdutil.AddFilenameOptionFlags(cmd, &o.FilenameOptions, "to use to create the resource")

    // 绑定命令参数：--validate
	cmdutil.AddValidateFlags(cmd)

    // 绑定命令参数：--edit
	cmd.Flags().BoolVar(&o.EditBeforeCreate, "edit", o.EditBeforeCreate, "Edit the API resource before creating")

    // 绑定命令参数：--windows-line-endings
	cmd.Flags().Bool("windows-line-endings", runtime.GOOS == "windows",
                     "Only relevant if --edit=true. Defaults to the line ending native to your platform.")

    // 绑定命令参数：--save-config=false，如果为 true 将对象的配置保存在注解中
	cmdutil.AddApplyAnnotationFlags(cmd)

    // 绑定命令参数：--dry-run=none，如果为 client 则只会打印对象，并不会实际生成
	cmdutil.AddDryRunFlag(cmd)

    // 绑定命令参数：--selector
	cmd.Flags().StringVarP(&o.Selector, "selector", "l", o.Selector,
                           "Selector (label query) to filter on, supports '=', '==', and '!='.(e.g. -l key1=value1,key2=value2)")
    // 绑定命令参数：--raw
	cmd.Flags().StringVar(&o.Raw, "raw", o.Raw, "Raw URI to POST to the server.  Uses the transport specified by the kubeconfig file.")

    // 绑定命令参数：--field-manager
	cmdutil.AddFieldManagerFlagVar(cmd, &o.fieldManager, "kubectl-create")

    // 绑定命令参数：--output
	o.PrintFlags.AddFlags(cmd)

	// 添加 kubectl create 的子命令:
    // namespace/quota/secret/configmap/serviceaccount/service/deployment/...
	cmd.AddCommand(NewCmdCreateNamespace(f, ioStreams))
	cmd.AddCommand(NewCmdCreateQuota(f, ioStreams))
	cmd.AddCommand(NewCmdCreateSecret(f, ioStreams))
	cmd.AddCommand(NewCmdCreateConfigMap(f, ioStreams))
	cmd.AddCommand(NewCmdCreateServiceAccount(f, ioStreams))
	cmd.AddCommand(NewCmdCreateService(f, ioStreams))
	cmd.AddCommand(NewCmdCreateDeployment(f, ioStreams))
	cmd.AddCommand(NewCmdCreateClusterRole(f, ioStreams))
	cmd.AddCommand(NewCmdCreateClusterRoleBinding(f, ioStreams))
	cmd.AddCommand(NewCmdCreateRole(f, ioStreams))
	cmd.AddCommand(NewCmdCreateRoleBinding(f, ioStreams))
	cmd.AddCommand(NewCmdCreatePodDisruptionBudget(f, ioStreams))
	cmd.AddCommand(NewCmdCreatePriorityClass(f, ioStreams))
	cmd.AddCommand(NewCmdCreateJob(f, ioStreams))
	cmd.AddCommand(NewCmdCreateCronJob(f, ioStreams))
	cmd.AddCommand(NewCmdCreateIngress(f, ioStreams))
	return cmd
}
```

### kubectl create 子命令

kubectl create 有非常多的子命令，并且这些子命令有较多复用。

[pkg/kubectl/cmd/create/create.go]

子命令公共结构体：

```go
// NewCreateSubcommandOptions returns initialized CreateSubcommandOptions
func NewCreateSubcommandOptions(ioStreams genericclioptions.IOStreams) *CreateSubcommandOptions {
	return &CreateSubcommandOptions{
		PrintFlags: genericclioptions.NewPrintFlags("created").WithTypeSetter(scheme.Scheme),
		IOStreams:  ioStreams,
	}
}

// CreateSubcommandOptions is an options struct to support create subcommands
type CreateSubcommandOptions struct {
	// PrintFlags holds options necessary for obtaining a printer
	PrintFlags *genericclioptions.PrintFlags

	// Name of resource being created
	Name string

	// StructuredGenerator is the resource generator for the object being created
	StructuredGenerator generate.StructuredGenerator

	// DryRun is true if the command should be simulated but not run against the server
	DryRun           bool
	CreateAnnotation bool

	Namespace        string
	EnforceNamespace bool

	Mapper        meta.RESTMapper
	DynamicClient dynamic.Interface

	PrintObj printers.ResourcePrinterFunc

	genericclioptions.IOStreams
}
```

子命令的通用参数补全方法，所有 create 子命令都会调用：

```go
// 补全所有的子命令上下文参数
func (o *CreateSubcommandOptions) Complete(f cmdutil.Factory, cmd *cobra.Command, args []string, generator generate.StructuredGenerator) error {
	// kubectl create namespace helloworld
	// 获得资源名 name = helloworld
	name, err := NameFromCommandArgs(cmd, args)
	if err != nil {
		return err
	}

	o.Name = name
	o.StructuredGenerator = generator
	o.DryRun = cmdutil.GetDryRunFlag(cmd)
	o.CreateAnnotation = cmdutil.GetFlagBool(cmd, cmdutil.ApplyAnnotationsFlag)

	if o.DryRun {
		o.PrintFlags.Complete("%s (dry run)")
	}
	printer, err := o.PrintFlags.ToPrinter()
	if err != nil {
		return err
	}

	o.PrintObj = func(obj kruntime.Object, out io.Writer) error {
		return printer.PrintObj(obj, out)
	}

	o.Namespace, o.EnforceNamespace, err = f.ToRawKubeConfigLoader().Namespace()
	if err != nil {
		return err
	}

	// 使用 kubectl config 生成 dynamic client
	o.DynamicClient, err = f.DynamicClient()
	if err != nil {
		return err
	}

	o.Mapper, err = f.ToRESTMapper()
	if err != nil {
		return err
	}

	return nil
}
```

子命令通用的资源创建：

```go
// Run executes a create subcommand using the specified options
func (o *CreateSubcommandOptions) Run() error {
	obj, err := o.StructuredGenerator.StructuredGenerate()
	if err != nil {
		return err
	}

	if o.DryRun {
		if meta, err := meta.Accessor(obj); err == nil && o.EnforceNamespace {
			meta.SetNamespace(o.Namespace)
		}
		return o.PrintObj(obj, o.Out)
	}

	// create subcommands have compiled knowledge of things they create, so type them directly
	gvks, _, err := scheme.Scheme.ObjectKinds(obj)
	if err != nil {
		return err
	}
	gvk := gvks[0]
	mapping, err := o.Mapper.RESTMapping(schema.GroupKind{Group: gvk.Group, Kind: gvk.Kind}, gvk.Version)
	if err != nil {
		return err
	}

	if err := kubectl.CreateOrUpdateAnnotation(o.CreateAnnotation, obj, scheme.DefaultJSONEncoder()); err != nil {
		return err
	}

	asUnstructured := &unstructured.Unstructured{}

	if err := scheme.Scheme.Convert(obj, asUnstructured, nil); err != nil {
		return err
	}
	if mapping.Scope.Name() == meta.RESTScopeNameRoot {
		o.Namespace = ""
	}
	actualObject, err := o.DynamicClient.Resource(mapping.Resource).Namespace(o.Namespace).Create(asUnstructured, metav1.CreateOptions{})
	if err != nil {
		return err
	}

	// ensure we pass a versioned object to the printer
	obj = actualObject

	return o.PrintObj(obj, o.Out)
}
```

#### kubectl create namespace 命令实现

[pkg/kubectl/cmd/create/create_namespace.go](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/create/create_namespace.go)

```go
func NewCmdCreateNamespace(f cmdutil.Factory, ioStreams genericclioptions.IOStreams) *cobra.Command {
	// Create Namespace 的命令行选项上下文缓存
	options := &NamespaceOpts{
		CreateSubcommandOptions: NewCreateSubcommandOptions(ioStreams),
	}

	cmd := &cobra.Command{
		Use:                   "namespace NAME [--dry-run]",
		DisableFlagsInUseLine: true,
		Aliases:               []string{"ns"},
		Short:                 i18n.T("Create a namespace with the specified name"),
		Long:                  namespaceLong,
		Example:               namespaceExample,
		Run: func(cmd *cobra.Command, args []string) {
			cmdutil.CheckErr(options.Complete(f, cmd, args))
			cmdutil.CheckErr(options.Run())
		},
	}

	options.CreateSubcommandOptions.PrintFlags.AddFlags(cmd)

	// 支持命令函参数：--save-config
	cmdutil.AddApplyAnnotationFlags(cmd)

	// 支持命令函参数：--validate
	cmdutil.AddValidateFlags(cmd)

	// 支持命令函参数：--generator / --dry-run
	cmdutil.AddGeneratorFlags(cmd, generateversioned.NamespaceV1GeneratorName)

	return cmd
}

// Complete completes all the required options
func (o *NamespaceOpts) Complete(f cmdutil.Factory, cmd *cobra.Command, args []string) error {
	name, err := NameFromCommandArgs(cmd, args)
	if err != nil {
		return err
	}

	var generator generate.StructuredGenerator
	switch generatorName := cmdutil.GetFlagString(cmd, "generator"); generatorName {
	case generateversioned.NamespaceV1GeneratorName:
		generator = &generateversioned.NamespaceGeneratorV1{Name: name}
	default:
		return errUnsupportedGenerator(cmd, generatorName)
	}

	// 补全 Create 子命令参数
	return o.CreateSubcommandOptions.Complete(f, cmd, args, generator)
}

// Run calls the CreateSubcommandOptions.Run in NamespaceOpts instance
// 使用 create 子命令公共方法创建 namespace
func (o *NamespaceOpts) Run() error {
	return o.CreateSubcommandOptions.Run()
}
```

#### kubectl create service

```go
// NewCmdCreateService is a macro command to create a new service
func NewCmdCreateService(f cmdutil.Factory, ioStreams genericclioptions.IOStreams) *cobra.Command {
	cmd := &cobra.Command{
		Use:     "service",
		Aliases: []string{"svc"},
		Short:   i18n.T("Create a service using specified subcommand."),
		Long:    "Create a service using specified subcommand.",
		Run:     cmdutil.DefaultSubCommandRun(ioStreams.ErrOut),
	}
	cmd.AddCommand(NewCmdCreateServiceClusterIP(f, ioStreams))
	cmd.AddCommand(NewCmdCreateServiceNodePort(f, ioStreams))
	cmd.AddCommand(NewCmdCreateServiceLoadBalancer(f, ioStreams))
	cmd.AddCommand(NewCmdCreateServiceExternalName(f, ioStreams))

	return cmd
}
```

## Kubectl util 实现

### Kubectl Config Flags

Kubectl 的配置文件

```go
var defaultCacheDir = filepath.Join(homedir.HomeDir(), ".kube", "http-cache")

// ConfigFlags 包含获取 REST 客户端配置所需的一组值
type ConfigFlags struct {
	CacheDir   *string
	KubeConfig *string

	// config flags
	ClusterName      *string
	AuthInfoName     *string
	Context          *string
	Namespace        *string
	APIServer        *string
	Insecure         *bool
	CertFile         *string
	KeyFile          *string
	CAFile           *string
	BearerToken      *string
	Impersonate      *string
	ImpersonateGroup *[]string
	Username         *string
	Password         *string
	Timeout          *string

	clientConfig clientcmd.ClientConfig
	lock         sync.Mutex
	// If set to true, will use persistent client config and
	// propagate the config to the places that need it, rather than
	// loading the config multiple times
	usePersistentConfig bool
}

// NewConfigFlags 创建配置文件选项，并给予默认值
func NewConfigFlags(usePersistentConfig bool) *ConfigFlags {
	impersonateGroup := []string{}
	insecure := false

	return &ConfigFlags{
		Insecure:   &insecure,
		Timeout:    stringptr("0"),
		KubeConfig: stringptr(""),

		CacheDir:         stringptr(defaultCacheDir),
		ClusterName:      stringptr(""),
		AuthInfoName:     stringptr(""),
		Context:          stringptr(""),
		Namespace:        stringptr(""),
		APIServer:        stringptr(""),
		CertFile:         stringptr(""),
		KeyFile:          stringptr(""),
		CAFile:           stringptr(""),
		BearerToken:      stringptr(""),
		Impersonate:      stringptr(""),
		ImpersonateGroup: &impersonateGroup,

		usePersistentConfig: usePersistentConfig,
	}
}

// WithDeprecatedPasswordFlag 启用用户名和密码配置 Flags（这是两个已经弃用的 Flags）
func (f *ConfigFlags) WithDeprecatedPasswordFlag() *ConfigFlags {
	f.Username = stringptr("")
	f.Password = stringptr("")
	return f
}

// AddFlags 将配置文件的相关内容绑定到命令行选项上
func (f *ConfigFlags) AddFlags(flags *pflag.FlagSet) {
	// --kubeconfig 指定配置文件路径
	if f.KubeConfig != nil {
		flags.StringVar(f.KubeConfig, "kubeconfig", *f.KubeConfig, "Path to the kubeconfig file to use for CLI requests.")
	}
	if f.CacheDir != nil {
		flags.StringVar(f.CacheDir, flagHTTPCacheDir, *f.CacheDir, "Default HTTP cache directory")
	}

	// add config options
	if f.CertFile != nil {
		flags.StringVar(f.CertFile, flagCertFile, *f.CertFile, "Path to a client certificate file for TLS")
	}
	if f.KeyFile != nil {
		flags.StringVar(f.KeyFile, flagKeyFile, *f.KeyFile, "Path to a client key file for TLS")
	}
	if f.BearerToken != nil {
		flags.StringVar(f.BearerToken, flagBearerToken, *f.BearerToken, "Bearer token for authentication to the API server")
	}
	if f.Impersonate != nil {
		flags.StringVar(f.Impersonate, flagImpersonate, *f.Impersonate, "Username to impersonate for the operation")
	}
	if f.ImpersonateGroup != nil {
		flags.StringArrayVar(f.ImpersonateGroup, flagImpersonateGroup, *f.ImpersonateGroup,
							 "Group to impersonate for the operation, this flag can be repeated to specify multiple groups.")
	}
	if f.Username != nil {
		flags.StringVar(f.Username, flagUsername, *f.Username, "Username for basic authentication to the API server")
	}
	if f.Password != nil {
		flags.StringVar(f.Password, flagPassword, *f.Password, "Password for basic authentication to the API server")
	}
	if f.ClusterName != nil {
		flags.StringVar(f.ClusterName, flagClusterName, *f.ClusterName, "The name of the kubeconfig cluster to use")
	}
	if f.AuthInfoName != nil {
		flags.StringVar(f.AuthInfoName, flagAuthInfoName, *f.AuthInfoName, "The name of the kubeconfig user to use")
	}
	if f.Namespace != nil {
		flags.StringVarP(f.Namespace, flagNamespace, "n", *f.Namespace, "If present, the namespace scope for this CLI request")
	}
	if f.Context != nil {
		flags.StringVar(f.Context, flagContext, *f.Context, "The name of the kubeconfig context to use")
	}

	if f.APIServer != nil {
		flags.StringVarP(f.APIServer, flagAPIServer, "s", *f.APIServer, "The address and port of the Kubernetes API server")
	}
	if f.Insecure != nil {
		flags.BoolVar(f.Insecure, flagInsecure, *f.Insecure, "If true, the server's certificate will not be checked for validity. This will make your HTTPS connections insecure")
	}
	if f.CAFile != nil {
		flags.StringVar(f.CAFile, flagCAFile, *f.CAFile, "Path to a cert file for the certificate authority")
	}
	if f.Timeout != nil {
		flags.StringVar(f.Timeout, flagTimeout, *f.Timeout, "The length of time to wait before giving up on a single server request. Non-zero values should contain a corresponding time unit (e.g. 1s, 2m, 3h). A value of zero means don't timeout requests.")
	}

}

// ToRawKubeConfigLoader 返回 clientcmd.clientConfig
// clientcmd.clientConfig 是 client-go 中的内容
func (f *ConfigFlags) ToRawKubeConfigLoader() clientcmd.ClientConfig {
	if f.usePersistentConfig {
		return f.toRawKubePersistentConfigLoader()
	}
	return f.toRawKubeConfigLoader()
}

// ToRESTMapper 返回一个 RESTMapper，这是一个用于将输入内容转换为 REST 资源的方法
func (f *ConfigFlags) ToRESTMapper() (meta.RESTMapper, error) {
	// 使用 kubectl config 得到一个 Discovery Client
	discoveryClient, err := f.ToDiscoveryClient()
	if err != nil {
		return nil, err
	}

	// restmapper 是 client-go 的内容
	mapper := restmapper.NewDeferredDiscoveryRESTMapper(discoveryClient)
	expander := restmapper.NewShortcutExpander(mapper, discoveryClient)
	return expander, nil
}
```

### Kubectl Factory 实现

```go
func NewFactory(clientGetter genericclioptions.RESTClientGetter) Factory {
	if clientGetter == nil {
		panic("attempt to instantiate client_access_factory with nil clientGetter")
	}

	f := &factoryImpl{
		clientGetter: clientGetter,
	}

	return f
}

// <staging/src/k8s.io/cli-runtime/pkg/genericclioptions/config_flags.go>
// RESTClientGetter 是 ConfigFlags 的接口，用于提供更简单的方法来模拟命令并消除与结构类型的直接耦合。 
type RESTClientGetter interface {
	// ToRESTConfig returns restconfig
	ToRESTConfig() (*rest.Config, error)

	// ToDiscoveryClient returns discovery client
	ToDiscoveryClient() (discovery.CachedDiscoveryInterface, error)

	// ToRESTMapper returns a restmapper
	ToRESTMapper() (meta.RESTMapper, error)

	// ToRawKubeConfigLoader return kubeconfig loader as-is
	ToRawKubeConfigLoader() clientcmd.ClientConfig
}
```

Kubectl Factory 实现的接口：

```go
func (f *factoryImpl) ToRawKubeConfigLoader() clientcmd.ClientConfig {
	return f.clientGetter.ToRawKubeConfigLoader()
}
```

## Kubectl 扩展机制
