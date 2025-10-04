FOr future updates try to merge in the changes from the original repo and update the build process as needed.  Keep the sample-buildproject.yaml and template.yaml files as they are.

aws codebuild start-build --project-name sharp-heic-lambda-layer --region us-west-2 --profile hughesWeb2     


What I did to get this built using CodeBuild.  No idea why SAM commands aren't used but this is what Claude told me...

0. Fork the new repo into my github account AND set up the local GH repo to point to this origin. 
Ex. git remote set-url origin https://github.com/stokaace/sharp-heic-lambda-layer5.git

1. Set up bucket sharp-layer-heic-v5 for code to live in (or reuse bucket???)
2. Set up Role SharpHEICCodeBuildRole (or reuse this one?)

3. Gave hughesWeb2 premission to run codebuild by attaching custom policy: CodeBuildPOLICY (in AWS)

4. In template.yaml rename the export name in Outputs so it doesn't conflict with the previous version...
    Outputs:
        SharpHEICLayerArn:
            Value: !Ref SharpHEICLayer
            Description: ARN of the Sharp HEIC Lambda Layer v5
            Export:
            Name: SharpHEICLayerArnV5

5.Change the Source to the new forked Github repo in sample-buildproject.yaml

6. Change the stack name Value to something unique to not conflict with the previous version in EnvironmentVariables to the sample-buildproject.yaml...
        EnvironmentVariables:
          - Name: STACK_NAME
            Type: PLAINTEXT
            Value: sharp-heic-lambda-layer-v5

7. Deploy with this...

aws cloudformation deploy \
  --template-file sample-buildproject.yaml \
  --stack-name sharp-heic-codebuild-v5 \
  --parameter-overrides \
    CodeBuildRole=arn:aws:iam::990816207588:role/SharpHEICCodeBuildRole \
    DeploymentBucket=sharp-layer-heic-v5 \
    SourceRepo=https://github.com/stokaace/sharp-heic-lambda-layer5 \
  --region us-west-2 \
  --profile hughesWeb2
 
 8. Then run the build with this...
aws codebuild start-build --project-name sharp-heic-lambda-layer --region us-west-2 --profile hughesWeb2                 



------------------------
------------------------
------------------------



# Sharp for AWS Lambda (with HEIC support)
AWS Lambda Layer providing [sharp](https://github.com/lovell/sharp) with HEIC (and WebP) support

![Build Status](https://codebuild.us-east-1.amazonaws.com/badges?uuid=eyJlbmNyeXB0ZWREYXRhIjoiUnY2cHpCUEwybDl5b2lIUys2b1lhN1BxMFVvb1pxMU8rUUpNNG1hSEFFN2VCUmxkK2t6azZrMEVOY1Y2RW40TGZ3NlF1bUo1dUE0ZVhiRm5GN3Q2YlJBPSIsIml2UGFyYW1ldGVyU3BlYyI6IjVxNm9zL3pWa0dQa21lNXAiLCJtYXRlcmlhbFNldFNlcmlhbCI6MX0%3D&branch=main)

## Prerequisites

* Docker
* [SAM v1.33.0 or higher](https://github.com/awsdocs/aws-sam-developer-guide/blob/master/doc_source/serverless-sam-cli-install.md)
* Node v22 (for v5.x)

## Usage

Due to potential license concerns for the HEVC patent group, this repo can't be provided in the most convenient way which would be a shared lambda layer or an Application in the AWS Serverless Repo.

But you can compile and deploy this lambda layer yourself at your own risk and use it wihin your own accounts. All you need is an S3 bucket to deploy the compiled code to (replace `your-s3-bucket` in the code snippet below). Please see the note below regarding the build process.

It is recommended to automate this process using AWS CodeBuild. A buildspec file is provided in the repo. In that case you'll have to set the `SAM_BUCKET` environment variable in CodeBuild. For other environment variables see the table below. The base image that should be used is `aws/codebuild/amazonlinux2-x86_64-standard:5.0`.

A sample CloudFormation template is provided to setup the CodeBuild project, [sample-buildproject.yaml](sample-buildproject.yaml).

```bash
npm run build
SAM_BUCKET=your-s3-bucket npm run deploy
```


The example can be deployed using the following commands
```bash
cd examples
sam build
sam deploy --guided
```

### Lambda Layer
- Add the lambda layer with ARN `arn:aws:lambda:us-east-1:${AWS:AccountId}:layer:sharp-heic:${LAYER_VERSION}` to any lambda function (replace `${LAYER_VERSION}` with the appropriate version and `${AWS:AccountId}` if you're not using a layer from the same account as the function). You can also import the layer ARN using `!ImportValue SharpHEICLayerArn`.
- Remove sharp from the dependencies in the function code (it will otherwise conflict with the one provided through the layer)
- See [example template](examples/sam-template.yaml) for a complete sample template.

### Environment Variables for build
|            Name | Required |           Default Value |                                   Description |
|-----------------|----------|-------------------------|-----------------------------------------------|
|      SAM_BUCKET |      yes |                         | Name of S3 Bucket to store layer              |
|       S3_PREFIX |       no | sharp-heic-lambda-layer | Prefix within S3 Bucket to store layer        |
|      STACK_NAME |       no | sharp-heic-lambda-layer | Name of CloudFormation stack                  |
|      LAYER_NAME |       no |              sharp-heic | Name of layer                                 |
|      AWS_REGION |       no |               us-east-1 | AWS Region to deploy to                       |
| ORGANIZATION_ID |       no |                    none | ID of Organization to grant access to layer   |
|       PRINCIPAL |       no |                 account | Principal to grant access to layer            |

For details on `ORGANIZATION_ID` and `PRINCIPAL` please see the equivalent properties in the [CloudFormation Docs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-layerversionpermission.html).

The special value `none` for `ORGANIZATION_ID` is used to disable organization based access.
The special value `account` for `PRINCIPAL` is used to give access to the account the layer is deployed to.

The environment variables are used to create a `samconfig.toml` file that configures the `sam package` and `sam deploy` commands.

### Note regarding build process
Previously, some custom docker images were needed to build this layer. AWS has since published newer images which work out of the box. `saml-cli` version `v1.33.0` is using `public.ecr.aws/sam/build-nodejs14.x:latest-x86_64`

## Background
This repo exists as it is rather painful to compile all libraries required to get sharp to work with HEIC/HEIF files in an AWS Lambda environment. The sharp repository has several [issues](https://github.com/lovell/sharp/issues) related to this.

### Layer contents
This lambda layer contains the node module [sharp](https://github.com/lovell/sharp). But unlike a normal installation via `npm i sharp` this layer does not use the prebuilt sharp and libvips binaries. This layer compiles libwebp, libde265, libheif, libvips, and sharp from source in order to provide HEIC/HEIF (and webp) functionality in an AWS Lambda environment.

### Dependencies
The following table lists the release version of this repo together with the version of each dependency. Patch versions are related to the build process or documentation and have the same dependencies as the minor version.
| release |  sharp | libvips | libheif | libwebp | libde265  |   x265 | libaom | nodejs |
|---------|--------|---------|---------|---------|-----------|--------|--------|--------|
|   1.2.0 | 0.28.2 |  8.10.6 |  1.12.0 |   1.2.0 |    1.0.8  |      - |        |     12 |
|   1.1.0 | 0.27.0 |  8.10.5 |  1.10.0 |   1.1.0 |    1.0.8  |      - |        |     12 |
|   2.0.0 | 0.29.1 |  8.11.3 |  1.12.0 |   1.2.1 |    1.0.8  |      - |        |     14 |
|   3.0.0 | 0.30.7 |  8.12.2 |  1.12.0 |   1.2.4 |    1.0.8  |      - |        |     16 |
|   3.1.0 | 0.30.7 |  8.12.2 |  1.12.0 |   1.2.4 |    1.0.8  |      - |        |     16 |
|   3.2.0 | 0.30.7 |  8.12.2 |  1.12.0 |   1.3.2 |    1.0.12 |      - |        |     16 |
|   4.1.0 | 0.33.3 |  8.15.2 |  1.17.6 |   1.4.0 |    1.0.15 |    3.6 |        |     20 |
|   4.1.3 | 0.33.3 |  8.15.2 |  1.17.6 |   1.4.0 |    1.0.15 |    3.6 |        |     20 |
|   4.2.0 | 0.33.5 |  8.15.3 |  1.18.2 |   1.4.0 |    1.0.15 |    3.6 |  3.9.1 |     20 |
|   5.0.0 | 0.34.3 |  8.17.1 |  1.20.1 |   1.6.0 |    1.0.16 |    4.1 | 3.12.1 |     22 |

### CompatibleRuntimes
- `nodejs12.x` (v1.x)
- `nodejs14.x` (v2.x)
- `nodejs16.x` (v3.x)
- `nodejs20.x` (v4.x)
- `nodejs22.x` (v5.x)

## Contributions
If you would like to contribute to this repository, please open an issue or submit a PR.

You can also use the Sponsor button on the right if you'd like.

## Licenses
- libheif and libde265 are distributed under the terms of the GNU Lesser General Public License. Copyright Struktur AG. See https://github.com/strukturag/libheif/blob/master/COPYING and https://github.com/strukturag/libde265/blob/master/COPYING for details.
- x265 is free to use under the GNU GPL and is also available under a commercial license. See https://www.x265.org/ for details.
- libwebp is Copyright Google Inc. See https://github.com/webmproject/libwebp/blob/master/COPYING for details.
- sharp is licensed under the Apache License, Version 2.0. Copyright Lovell Fuller and contributors. See https://github.com/lovell/sharp/blob/master/LICENSE for details.
- libvips is licensed under the LGPL 2.1+. See https://github.com/libvips/libvips/blob/master/COPYING for details.
- libaom is subject to the terms of the BSD 2 Clause License and the Alliance for Open Media Patent License 1.0. See https://aomedia.googlesource.com/aom/#license-header
- The remainder of the code in this repository is licensed under the MIT License. See [LICENSE](LICENSE) for details.

## Related Resources
Visit [sharp.pixelplumbing.com](https://sharp.pixelplumbing.com/) for complete instructions on sharp.

### Code Repositories
- [sharp](https://github.com/lovell/sharp)
- [libheif](https://github.com/strukturag/libheif)
- [libde265](https://github.com/strukturag/libde265)
- [libwebp](https://github.com/webmproject/libwebp)
- [libvips](https://github.com/libvips/libvips)
- [libaom](https://aomedia.googlesource.com/aom/)
