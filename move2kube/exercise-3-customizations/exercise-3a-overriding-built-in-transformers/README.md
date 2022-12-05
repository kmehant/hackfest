# Customizing generated Dockerfile and behavior of built-in transformer

Lets first run the Move2Kube transform on `enterprise-app` without any customizations and observe the output:

```bash
$ move2kube transform -s ../../source/enterprise-app/ --qa-skip && ls myproject/source/frontend && cat myproject/source/frontend/Dockerfile && ls myproject/deploy && rm -rf myproject
```

```text
Dockerfile        __mocks__         jest.config.js    package-lock.json src               test-setup.js     webpack.dev.js
LICENSE           dist              manifest.yml      package.json      stories           tsconfig.json     webpack.prod.js
README.md         dr-surge.js       nodemon.json      server.js         stylePaths.js     webpack.common.js
#   Copyright IBM Corporation 2020
#
#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at
#
#        http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.

FROM registry.access.redhat.com/ubi8/ubi-minimal:latest
RUN microdnf update && microdnf install wget xz tar && microdnf clean all && \
    wget https://nodejs.org/dist/v14.20.0/node-v14.20.0-linux-x64.tar.xz && \
    tar -xJf node-v14.20.0-linux-x64.tar.xz && \
    mv node-v14.20.0-linux-x64 /node && \
    rm -f node-v14.20.0-linux-x64.tar.xz
ENV PATH="$PATH:/node/bin"
COPY . .
RUN npm install
RUN npm run build
USER root
RUN chown -R 1001:0 /opt/app-root/src/.npm
RUN chmod -R 775 /opt/app-root/src/.npm
USER 1001
EXPOSE 8080
CMD npm run start
cicd                  compose               knative               knative-parameterized yamls                 yamls-parameterized
```

* If we notice the Dockerfile generated for the frontend app, it uses registry.access.redhat.com/ubi8/nodejs-12 as base image
* There are no scripts named start-nodejs.sh in the frontend service directory
* The Kubernetes yamls are generated in myproject/deploy/yamls directory


Lets customize the ouputs. Specifically, lets : 

* change the base image `registry.access.redhat.com/ubi8/nodejs-12` to `quay.io/konveyor/nodejs-12` in the generated nodejs dockerfile.
* add a new `start-nodejs.sh` file to `frontend` service directory which will be used in the CMD dockerfile instruction.
* change the CMD instruction in the generated dockerfile to use the `start-nodejs.sh` file.
* change the default location of the kubernetes yamls that are generated from `myproject/deploy/yamls` to `myproject/yamls-elsewhere`

We will use a [custom configured version](https://github.com/konveyor/move2kube-transformers/tree/main/custom-dockerfile-change-built-in-behavior) of the [nodejs built-in transformer](https://github.com/konveyor/move2kube/tree/main/assets/built-in/transformers/dockerfilegenerator/nodejs) and [kubernetes built-in transformer](https://github.com/konveyor/move2kube/tree/main/assets/built-in/transformers/kubernetes/kubernetes) to achieve this. We copy [it](https://github.com/konveyor/move2kube-transformers/tree/main/custom-dockerfile-change-built-in-behavior) into the `customizations` sub-directory.

```bash
$ curl https://move2kube.konveyor.io/scripts/download.sh | bash -s -- -d custom-dockerfile-change-built-in-behavior -r move2kube-transformers -o customizations
```

Now lets transform with this customization. Specify the customization using the -c flag and see the customized output.

```bash
$ move2kube transform -s ../../source/enterprise-app/ -c customizations/ --qa-skip && ls myproject/source/frontend && cat myproject/source/frontend/Dockerfile && ls myproject/ && rm -rf myproject
```

```text
Dockerfile        __mocks__         jest.config.js    package-lock.json src               stylePaths.js     webpack.common.js
LICENSE           dist              manifest.yml      package.json      start-nodejs.sh   test-setup.js     webpack.dev.js
README.md         dr-surge.js       nodemon.json      server.js         stories           tsconfig.json     webpack.prod.js
#   Copyright IBM Corporation 2020
#
#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at
#
#        http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.

FROM quay.io/konveyor/nodejs-12
COPY . .
RUN npm install
RUN npm run build
EXPOSE 8080
CMD sh start-nodejs.sh
Readme.md                     scripts                       yamls-elsewhere
deploy                        source                        yamls-elsewhere-parameterized
```

* The Dockerfile generated for the frontend app contains the custom base image
* A new file named `start-nodejs.sh` is generated in the frontend directory
* The kubernetes yamls are now generated in `myproject/yamls-elsewhere` directory, and hence the parameterized yamls are also in `myproject/yamls-elsewhere-parameterized` directory.

Please refer the anotamy of the [custom transformers](https://move2kube.konveyor.io/tutorials/customizing-the-output/custom-dockerfile-change-built-in-behavior#anatomy-of-transformers-in-custom-dockerfile-change-built-in-behavior) that are used in this exercise for better understanding.
