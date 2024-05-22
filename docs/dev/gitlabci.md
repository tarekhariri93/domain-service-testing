# Gitlab CI

We are using gitlab CI to run basic validations on PRs

## Build container 

Unit tests are using [image](../../dist/images/Dockerfile.build) that contain all the dependencies in order to run the tests
If this image is updated, build needs to be done manually and then pushed to SDN docker domain registry.

```sh
cd ./dist/images
docker build -t gitlab-master.nvidia.com:5005/sdn/tools/ovn-domain-service-build:<verison> -f Dockerfile.build .
```

Before pushing the image, you need to run docker log in to nvidia server, create a token in [gitlab](https://gitlab-master.nvidia.com/-/user_settings/personal_access_tokens) and loging with this token

```sh
docker login -u=<user>@nvidia.com gitlab-master.nvidia.com:5005
```

Push the image

```sh
docker push gitlab-master.nvidia.com:5005/sdn/tools/ovn-domain-service-build:<version>
```

Push may require permissions and someone from SDN need to add you to SDN group

After the image is pushed need to update the [.gitlab-ci](../../.gitlab-ci.yml) to use the new image on the required jobs