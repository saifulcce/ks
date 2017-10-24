# Kubernetes series part 5

The objective here is to create a test environment - that is build the frontend and serve it in our python server as static resources.

To do this we will create a separate deployment and we will enhance our python server to take the environment into account.

1. navigate to ks5

    ```bash
    ➜ pwd
        ~/dev/github/santiaago/ks/ks5
    ```

1. start minikube

    ```bash
    ➜ minikube start
    ```

1. switch to minikube context

    ```bash
    ➜ eval $(minikube docker-env)
    ```

    If you ever need to switch back to your machine's context do:

    ```bash
    ➜ eval $(docker-machine env -u)
    ```

1. build web docker image

    ```bash
    ➜ docker build -f ./web/Dockerfile -t ks5webimage .
    ```

1. build web server docker image

    ```bash
    ➜ docker build -f ./server/Dockerfile -t ks5webserverimage .
    ```

1. mount ks5 source code

    In a _separate_ terminal, in the root of the project (this terminal needs to keep running the **whole** time you're debugging...).

    ```bash
    ➜ pwd
        ~/dev/github/santiaago/ks/ks5
    ➜ minikube mount .:/mounted-ks5-src
        Mounting ./app/src into /mounted-ks5-app-src on the minikube VM
        This daemon process needs to stay alive for the mount to still be accessible...
        ufs starting
    ```

    For more information about mounting volumes read these [docs](https://github.com/kubernetes/minikube/blob/master/docs/host_folder_mount.md)

1. add environment variable to webserver section in deployment yaml file

    As described in the [flask docs](http://flask.pocoo.org/docs/0.12/server/) we will add the `FLASK_DEBUG` environment variable to get the reloading experience we want.

    ```yaml
    command: ["python"]
    args: ["-m", "flask", "run"]
    env:
    - name: FLASK_DEBUG
        value: "1"
    ports:
    - containerPort: 5000
    ```

1. add the web server volumes to the deployment yaml file

    Update the global volume section of the `./config/dev.ks.deployment.yaml` with a `python-server volume that point to the python backend so that changes are picked up on save.

    ```yaml
    volumes:
    - name: python-server
    hostPath:
        path: /mounted-ks5-src/server
    - name: frontend
    hostPath:
        path: /mounted-ks5-src/app/src
    ```

    The reference it in the webserver image section

    ```yaml
    volumeMounts:
    - mountPath: /server
    name: python-server
    ```

1. create deployment and service

    ```bash
    ➜ kubectl create -f ./config/dev.ks.deployment.yaml
        deployment "ks5web" created

    ➜ kubectl create -f ./config/dev.ks.service.yaml
        service "ks5web" created
    ➜ kubectl get all
        NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
        deploy/ks5web   1         1         1            1           31s

        NAME                   DESIRED   CURRENT   READY     AGE
        rs/ks5web-2671084145   1         1         1         31s

        NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
        deploy/ks5web   1         1         1            1           31s

        NAME                         READY     STATUS    RESTARTS   AGE
        po/ks5web-2671084145-f0f71   2/2       Running   0          31s
    ```

1. get web server logs

    ```bash
    ➜ kubectl logs ks5web-2671084145-f0f71 ks5webserver
        * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
        * Restarting with stat
        /usr/local/lib/python3.5/runpy.py:125: RuntimeWarning: 'flask.cli' found in sys.modules after import of package 'flask', but prior to execution of 'flask.cli'; this may result in unpredictable behaviour
        warn(RuntimeWarning(msg))
        * Debugger is active!
        * Debugger PIN: 207-014-748
    ```

1. service ks5web

    ```bash
    ➜ minikube service ks5web --url
    ```

1. check backend updates by changing the `hello.py` controller

    If you attach the logs with `-f` and update `./server/controllers/hello.py` you should see

    ```bash
    Detected change in '/server/controllers/hello.py', reloading` in the logs.
    ```

    Full logs:
    ```bash
    ➜ kubectl logs ks5web-2671084145-gtz9q ks5webserver -f
        * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
        * Restarting with stat
        /usr/local/lib/python3.5/runpy.py:125: RuntimeWarning: 'flask.cli' found in sys.modules after import of package 'flask', but prior to execution of 'flask.cli'; this may result in unpredictable behaviour
        warn(RuntimeWarning(msg))
        * Debugger is active!
        * Debugger PIN: 207-014-748
        --------------------------------------------------------------------------------
        INFO in hello [/server/controllers/hello.py:8]:
        hello controller called
        --------------------------------------------------------------------------------
        127.0.0.1 - - [23/Oct/2017 09:53:50] "GET /api/hello HTTP/1.1" 200 -
        * Detected change in '/server/controllers/hello.py', reloading
        * Restarting with stat
        /usr/local/lib/python3.5/runpy.py:125: RuntimeWarning: 'flask.cli' found in sys.modules after import of package 'flask', but prior to execution of 'flask.cli'; this may result in unpredictable behaviour
        warn(RuntimeWarning(msg))
        * Debugger is active!
        * Debugger PIN: 207-014-748
        --------------------------------------------------------------------------------
        INFO in hello [/server/controllers/hello.py:8]:
        hello controller called
    ```

    If you then refresh the browser you should se the changes in the UI.

1. minikube mounted volume command should detect this changes

    ```bash
    ➜ minikube mount .:/mounted-ks5-src
        Mounting . into /mounted-ks5-src on the minikube VM
        This daemon process needs to stay alive for the mount to still be accessible...
        ufs starting
        Rename ./server/controllers/__pycache__/hello.cpython-35.pyc.140423133386000 to hello.cpython-35.pyc
        rel  results in server/controllers/__pycache__/hello.cpython-35.pyc
        rename ./server/controllers/__pycache__/hello.cpython-35.pyc.140423133386000 to server/controllers/__pycache__/hello.cpython-35.pyc gets <nil>
    ```