---
layout: post
title: Unit Test your Helm Charts
img: helm.png
tags: [kubernetes, helm]
---

I'm writing and maintaining a good amount of `Helm Charts` on a daily basis. As Helm charts can contains logic in it, complexity keeps increasing over time.  
The question now, is how to ensure charts don't introduce regressions or bugs over change?  
The answer is simple and we are already used to it, we need to use tests like we do for software!  
In this article, we will focus on unit tests.

## __Helm Unittest__
<br/>

### __Setup__

Thanks to the [helm unit test plugin](https://github.com/quintush/helm-unittest) from Quintush, we can easily create unit tests using `yaml` syntax.  

First things first, we need to install the plugin
```sh
> helm plugin install https://github.com/quintush/helm-unittest
```

We need then to create a folder `tests` under the helm chart root folder.  
Now our test(s) suite file will be placed under the tests and will have the `_test.yaml` suffix.
```
.
├── charts
├── Chart.yaml
├── templates
│   ├── _helpers.tpl
│   ├── rbac.yaml
│   └── sa.yaml
├── tests
│   └── rbac_test.yaml
└── values.yaml
```

We will create the `rbac_test.yaml` with the following:
```yaml
suite: service account tests
templates:
  - rbac.yaml
```
We are giving a description to our test suite and the list of `templates` yaml file we want to test.

### __Hands-on__

Let's take a look at our `rbac.yaml` file
```yaml
{% raw %}
{{- if .Values.rbac.create }}
{{- if .Values.rbac.scoped }}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
  - apiGroups:
      - ""
    resources:
      - secrets
    verbs:
      - get
      - watch
      - list
{{- else }}
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
  - apiGroups:
      - ""
    resources:
      - secrets
    verbs:
      - get
      - watch
      - list
{{- end }}
{{- end }}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-reader
subjects:
  - kind: ServiceAccount
    name: secret-reader-robot
    namespace: {{ .Release.Namespace }}
roleRef:
{{- if .Values.rbac.scoped }}
  kind: Role
{{- else }}
  kind: ClusterRole
{{- end }}
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
{% endraw %}
```

We have some logic to handle the fact we want or not create the `role` part of the RBAC and also if we want it scoped with a `Role` or cluster-wide with `ClusterRole`.  

It's time to get our hands dirty and write some tests for that file.  
We don't want introduce regressions in our future changes.  

We will follow the `Arrange-Act-Assert pattern` (AAA pattern), that will provide a uniform structure for all tests.

```yaml
suite: service account tests
templates:
  - rbac.yaml
tests:
  - it: Should render
    set:
      rbac:
        create: true
        scoped: false
    asserts:
      - hasDocuments:
          count: 2
      - isKind:
          of: ClusterRole
        documentIndex: 0
      - isKind:
          of: RoleBinding
        documentIndex: 1
```

- `set` section allows us to arrange the values we want use for the given test  
If some values are omitted, the corresponding in values file will be used, same if the `set` section is omitted  
- `asserts` section allows us to describe the expected output    
As it a list, we can have single or multiple asserts per test, but please do not create only one test with all possible assertion you want to test.  

The `Act` part of the pattern is done by rendering the chart. This is handle by the helm unittest plugin.

So, what this one is testing?  
</br>
For the given values:
- rbac.create = true
- rbac.scoped = false

We expect to have:
- Two documents rendered
- First one has a `ClusterRole` kind
- Second one has a `RoleBinding` kind

We are good to execute the `helm unittest` command to verify the test result
```sh  
# we are in the chart directory

> helm unittest . -3

### Chart [ unit_test_demo ] .

 PASS  service account tests	tests/rbac_test.yaml

Charts:      1 passed, 1 total
Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshot:    0 passed, 0 total
Time:        2.297962ms

```

Great! We have tests the default behavior we want to our chart, now we have to also check the other path of our logic.  
This time instead of overriding manually our values with the `set` section, we will use the `values` section. It will allow us to provide a values file.

```yaml
  - it: Should not render role as rbac.create is false
    set:
      rbac:
        create: false
        scoped: false
    asserts:
      - hasDocuments:
          count: 1
      - isKind:
          of: RoleBinding

  - it: Should render a Role kind as rbac.scoped is true
    values:
      - ./values/scoped_values.yaml
    asserts:
      - hasDocuments:
          count: 2
      - isKind:
          of: Role
        documentIndex: 0
```

Nothing new in the assert part, we are just testing other possibility of our logic.  

Let's run the tests again
```sh
> helm unittest . -3

### Chart [ unit_test_demo ] .

 PASS  service account tests	tests/rbac_test.yaml

Charts:      1 passed, 1 total
Test Suites: 1 passed, 1 total
Tests:       3 passed, 3 total
Snapshot:    0 passed, 0 total
Time:        3.383811ms
```

Everything is green, great!

Now, let's check the last part of our logic
```yaml
  - it: Should have a Role roleRef
    set:
      rbac:
        create: true
        scoped: true
    asserts:
      - equal:
          path: roleRef.kind
          value: Role
        documentIndex: 1
```

We are using the `equal` assert type that take the full `path` of the part we want to test and the desired `value`.

There is a lot more of `assertion types`, you can find the list [here](https://github.com/quintush/helm-unittest/blob/master/DOCUMENT.md#assertion-types)

Now, let's assume someone inverted by accident the logic for the `roleRef` part.

```sh
> helm unittest . -3

 FAIL  service account tests	tests/rbac_test.yaml
	- Should have a Role roleRef

		- asserts[0] `equal` fail
			Template:	unit_test_demo/templates/rbac.yaml
			DocumentIndex:	0
			Path:	roleRef.kind
			Expected to equal:
				Role
			Actual:
				ClusterRole
			Diff:
				--- Expected
				+++ Actual
				@@ -1,2 +1,2 @@
				-Role
				+ClusterRole


Charts:      1 failed, 0 passed, 1 total
Test Suites: 1 failed, 0 passed, 1 total
Tests:       1 failed, 3 passed, 4 total
Snapshot:    0 passed, 0 total
Time:        4.478972ms

Error: plugin "unittest" exited with error
```

We get a direct feedback of:
- Which test failed
- Which assert
- For what reason

This is sweet!  

### __More testing__

The `helm unittest` plugin has some other useful functionality
- `Snapshot testing`, documentation is [here](https://github.com/quintush/helm-unittest#snapshot-testing)  
This can be very useful to test the content of ConfigMaps for example
- Defining `Release` option globally on test suite or locally for specific test ([more info](https://github.com/quintush/helm-unittest/blob/master/DOCUMENT.md#test-suite))
- Subchart testing, [here](https://github.com/quintush/helm-unittest#dependent-subchart-testing)

### __Conclusion__

That very sweet, we are able to unit test our chart!  
Next step is to add a step in our `CI/CD pipeline` to run the tests on every push to the repository.  
That way we ensure there is no regression on our logic over changes.

The BDD style also allows everyone to have a better understanding of the logic and control flow of helm charts by only reading the tests suites.

### __Happy Helming!__