---
id: add_facilities_to_chaos_daemon
title: Add Facilities to Chaos Daemon
sidebar_label: Add Facilities to Chaos Daemon
---

In [Develop a new chaos](dev_hello_world.md), we have added a new chaos type named `HelloWorldChaos`, which will print `hello world` in `chaos-controller-manager`. To actually run the chaos, we need to configure some facilities for Chaos Daemon - so that `controller-manager` can select the specified Pods according to the chaos configuration and sends the chaos request to the `chaos-daemon` corresponding to these Pods. Once these are done, the `chaos-daemon` could run the chaos at last.

This guide covers the following steps:

- [Add selector for HelloWorldChaos](#add-selector-for-helloworldChaos)
- [Implement the gRPC interface](#implement-grpc-interface-for-chaos-request)
- [Verify your chaos](#verify-your-chaos)

## Add selector for HelloWorldChaos

In Chaos Mesh, we have defined the `spec.selector` field to specify the scope of the chaos by namespace, labels, annotation, etc. You can refer to [Define the Scope of Chaos Experiment](../user_guides/experiment_scope.md) for more information. To specify the Pods for `HelloWorld` chaos:

1. Add the `Spec` field in `HelloWorldChaos`:

    ```golang
    // HelloWorldChaos is the Schema for the helloworldchaos API
    type HelloWorldChaos struct {
        metav1.TypeMeta   `json:",inline"`
        metav1.ObjectMeta `json:"metadata,omitempty"`

        // Spec defines the behavior of a pod chaos experiment
        Spec HelloWorldSpec `json:"spec"`
    }

    type HelloWorldSpec struct {
        Selector SelectorSpec `json:"selector"`
    }

    // GetSelector is a getter for Selector (for implementing SelectSpec)
    func (in *HelloWorldSpec) GetSelector() SelectorSpec {
        return in.Selector
    }
    ```

2. Generate boilerplate functions for the `spec` field. This is required to integrate the resource in Chaos Mesh.

    ```bash
    make generate
    ```

## Implement the gRPC interface

In order for `chaos-daemon` to accept requests from `chaos-controller-manager`, a new gRPC  interface is required for `chaos-controller-manager` and `chaos-daemon`. Take the steps below to add the gRPC interface:

1. Add the RPC in [chaosdaemon.proto](https://github.com/chaos-mesh/chaos-mesh/blob/master/pkg/chaosdaemon/pb/chaosdaemon.proto).

    ```proto
    service chaosDaemon {
        ...

        rpc ExecHelloWorldChaos(ExecHelloWorldRequest) returns (google.protobuf.Empty) {}
    }

    message ExecHelloWorldRequest {
        string container_id = 1;
    }
    ```

    You will need to update golang code generated by this proto file:

    ```bash
    make proto
    ```

2. Implement the gRPC service in `chaos-daemon`.

    Add a new file named `helloworld_server.go` under [chaosdaemon](https://github.com/chaos-mesh/chaos-mesh/tree/master/pkg/chaosdaemon), with the content as below:

    ```golang
    package chaosdaemon

    import (
        "context"
        "fmt"

        "github.com/golang/protobuf/ptypes/empty"

        pb "github.com/chaos-mesh/chaos-mesh/pkg/chaosdaemon/pb"
    )

    func (s *daemonServer) ExecHelloWorldChaos(ctx context.Context, req *pb.ExecHelloWorldRequest) (*empty.Empty, error) {
        log.Info("ExecHelloWorldChaos", "request", req)

        pid, err := s.crClient.GetPidFromContainerID(ctx, req.ContainerId)
        if err != nil {
            return nil, err
        }

        cmd := defaultProcessBuilder("sh", "-c", fmt.Sprintf("echo 'hello' `hostname`")).
            SetUtsNS(GetNsPath(pid, utsNS)).
            Build(context.Background())
        out, err := cmd.Output()
        if err != nil {
            return nil, err
        }
        if len(out) != 0 {
            log.Info("cmd output", "output", string(out))
        }

        return &empty.Empty{}, nil
    }
    ```

    After `chaos-daemon` receives the `ExecHelloWorldChaos` request, `chaos-daemon` will print `hello` to this container's hostname.

3. Send gRPC requests in reconcile.

    When a CRD object is updated (for example: create or delete), we need to compare the state specified in the object against the actual state, and then perform operations to make the actual cluster state reflect the state specified. This process is called `reconcile`. For `HelloworldChaos`, `chaos-controller-manager` needs to send chaos request to `chaos-daemon` in `reconcile`. To do this:

    1. Under [controllers](https://github.com/chaos-mesh/chaos-mesh/tree/master/controllers), create a directory named `helloworldchaos`, which includes a file named  `type.go` with the content as below:
    
        ```golang
        package helloworldchaos

        import (
            "context"
            "errors"
            "fmt"

            "github.com/go-logr/logr"
            "k8s.io/client-go/tools/record"
            ctrl "sigs.k8s.io/controller-runtime"
            "sigs.k8s.io/controller-runtime/pkg/client"

            "github.com/chaos-mesh/chaos-mesh/api/v1alpha1"
            "github.com/chaos-mesh/chaos-mesh/controllers/common"
            pb "github.com/chaos-mesh/chaos-mesh/pkg/chaosdaemon/pb"
            "github.com/chaos-mesh/chaos-mesh/pkg/utils"
        )

        type Reconciler struct {
            client.Client
            Log logr.Logger
        }

        // Reconcile reconciles a HelloWorldChaos resource
        func (r *Reconciler) Reconcile(req ctrl.Request, chaos *v1alpha1.HelloWorldChaos) (ctrl.Result, error) {
            r.Log.Info("Reconciling helloworld chaos")

            err := r.Apply(context.Background(), req, chaos)
            return ctrl.Result{}, err
        }

        // Apply applies helloworld chaos
        func (r *Reconciler) Apply(ctx context.Context, req ctrl.Request, chaos v1alpha1.InnerObject) error {
            r.Log.Info("Apply helloworld chaos")
            helloworldchaos, ok := chaos.(*v1alpha1.HelloWorldChaos)
            if !ok {
                return errors.New("chaos is not helloworldchao s")
            }

            pods, err := utils.SelectPods(ctx, r.Client, helloworldchaos.Spec.GetSelector())
            if err != nil {
                return err
            }

            for _, pod := range pods {
                daemonClient, err := utils.NewChaosDaemonClient(ctx, r.Client,
                    &pod, common.ControllerCfg.ChaosDaemonPort)
                if err != nil {
                    r.Log.Error(err, "get chaos daemon client")
                    return err
                }
                defer daemonClient.Close()
                if len(pod.Status.ContainerStatuses) == 0 {
                    return fmt.Errorf("%s %s can't get the state of container", pod.Namespace, pod.Name)
                }

                containerID := pod.Status.ContainerStatuses[0].ContainerID

                _, err = daemonClient.ExecHelloWorldChaos(ctx, &pb.ExecHelloWorldRequest{
                    ContainerId: containerID,
                })
                if err != nil {
                    return err
                }
            }

            return nil
        }

        // Recover means the reconciler recovers the chaos action
        func (r *Reconciler) Recover(ctx context.Context, req ctrl.Request, chaos v1alpha1.InnerObject) error {
            return nil
        }

        // Object would return the instance of chaos
        func (r *Reconciler) Object() v1alpha1.InnerObject {
            return &v1alpha1.HelloWorldChaos{}
        }
        ```

    > **Notes:**
    >
    > In our case here, the `Recover` function does nothing because `HelloWorldChaos` only prints some log and doesn't change anything. You may need to implement the `Recover` function in your development.

    2. Modify the `controllers/helloworldchaos_controller.go` file created in [Dev a new chaos](dev_hello_world.md), the `Reconcile` function updated to below:

        ```golang
        // +kubebuilder:rbac:groups=chaos-mesh.org,resources=helloworldchaos,verbs=get;list;watch;create;update;patch;delete
        // +kubebuilder:rbac:groups=chaos-mesh.org,resources=helloworldchaos/status,verbs=get;update;patch

        func (r *HelloWorldChaosReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
            logger := r.Log.WithValues("reconciler", "helloworldchaos")
            logger.Info("Hello World!")

            reconciler := helloworldchaos.Reconciler{
                Client: r.Client,
                Log: logger,
            }
            chaos := &v1alpha1.HelloWorldChaos{}
            if err := r.Get(context.Background(), req.NamespacedName, chaos); err != nil {
                return ctrl.Result{}, nil
            }

            result, err := reconciler.Reconcile(req, chaos)
            if err != nil {
                return ctrl.Result{}, nil
            }

            return result, nil
        }
        ```

## Verify your chaos

Now you are all set. It's time to verify the chaos type you just created. Take the steps below:

1. Make the Docker image. Refer to [Make the Docker image](dev_hello_world.md#make-the-docker-image).

2. Upgrade Chaos Mesh. Since we have already installed Chaos Mesh in [Develop a New Chaos](dev_hello_world.md#run-chaos), we only need to upgrade it:

    ```bash
    helm upgrade chaos-mesh helm/chaos-mesh --namespace=chaos-testing --set controllerManager. imagePullPolicy=Always --set chaosDaemon.imagePullPolicy=Always
    ```

3. Deploy the Pods for test:

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/chaos-mesh/apps/master/ping/busybox-statefulset.yaml
    ```

    This command deploys two Pods in the `busybox` namespace.

4. Create the chaos YAML file:

    ```yaml
    apiVersion: chaos-mesh.org/v1alpha1
    kind: HelloWorldChaos
    metadata:
      name: busybox-helloworld-chaos
    spec:
      selector:
        namespaces:
          - busybox
    ```

5. Apply the chaos:

    ```bash
    kubectl apply -f /path/to/helloworld.yaml
    ```
6. Verify your chaos. There are different logs to check to see whether your chaos works as expected:


    - Check the log of `chaos-controller-manager`:

        ```bash
        kubectl logs chaos-controller-manager-{pod-post-fix} -n chaos-testing
        ```

        The log is as follows:

        ```log
        2020-09-09T09:13:36.018Z        INFO    controllers.HelloWorldChaos     Reconciling helloworld chaos    {"reconciler": "helloworldchaos"}
        2020-09-09T09:13:36.018Z        INFO    controllers.HelloWorldChaos     Apply helloworld chaos  {"reconciler": "helloworldchaos"}
         ```

    - Check the log of `chaos-daemon`:

        ```bash
        kubectl logs chaos-daemon-{pod-post-fix} -n chaos-testing
        ```

        The log is as follows:

        ```log
        2020-09-09T09:13:36.036Z        INFO    chaos-daemon-server     exec hello world chaos  {"request": "container_id:\"docker://8f2918ee05ed587f7074a923cede3bbe5886277faca95d989e513f7b7e831da5\" "}
        2020-09-09T09:13:36.044Z        INFO    chaos-daemon-server     build command   {"command": "nsenter -u/proc/45664/ns/uts -- sh -c echo 'hello' `hostname`"}
        2020-09-09T09:13:36.058Z        INFO    chaos-daemon-server     cmd output      {"output": "hello busybox-1\n"}
        2020-09-09T09:13:36.064Z        INFO    chaos-daemon-server     exec hello world chaos  {"request": "container_id:\"docker://53e982ba5593fa87648edba665ba0f7da3f58df67f8b70a1354ca00447c00524\" "}
        2020-09-09T09:13:36.066Z        INFO    chaos-daemon-server     build command   {"command": "nsenter -u/proc/45620/ns/uts -- sh -c echo 'hello' `hostname`"}
        2020-09-09T09:13:36.070Z        INFO    chaos-daemon-server     cmd output      {"output": "hello busybox-0\n"}
        ```

    We can see the `chaos-daemon` prints `hello` to these two Pods.