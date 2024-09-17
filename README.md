# S3 File Uploader with Signed URLs
##cOverview
This project provides a file uploader built using AWS SAM that enables users to upload images to S3 through a presigned URL. The user selects an image through the front-end interface, and a Lambda function generates a signed URL for the upload. This ensures secure and controlled access to the S3 bucket, even from the client side.

## Features
### Vue.js Front-End: 
A simple front-end interface for selecting and uploading images.
### S3 Signed URL: 
A Lambda function generates a presigned URL that allows the user to upload directly to the S3 bucket.
### S3 Bucket: 
The uploaded files are stored in an S3 bucket, with CORS configured to allow client-side uploads.
### HTTP API Gateway: 
API Gateway is used to trigger the Lambda function that generates the presigned URL.
## Architecture
### S3 Bucket: 
An S3 bucket is created to store the uploaded images.
### Lambda Function: 
The Lambda function generates a presigned URL for the S3 bucket, allowing secure upload access.
### HTTP API Gateway: 
API Gateway is used to invoke the Lambda function that generates the presigned URL.
### Vue.js Client: 
The front-end interface allows users to upload images directly to the S3 bucket using the presigned URL.
### SAM Template
The AWS SAM template defines all the required resources including the Lambda function, S3 bucket, and HTTP API Gateway.

## SAM Template
```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: AWS::Serverless-2016-10-31
Description: S3 Uploader - sample application

Parameters:
  Auth0issuer:
    Type: String
    Description: The issuer URL from your Auth0 account.
    Default: https://dev-abcdefg.auth0.com/

Resources:
  # HTTP API
  MyApi:
    Type: AWS::Serverless::HttpApi
    Properties:
      Auth:
        Authorizers:
          MyAuthorizer:
            JwtConfiguration:
              issuer: !Ref Auth0issuer
              audience:
                - https://auth0-jwt-authorizer
            IdentitySource: "$request.header.Authorization"
        DefaultAuthorizer: MyAuthorizer
      CorsConfiguration:
        AllowMethods:
          - GET
          - POST
          - DELETE
          - OPTIONS
        AllowHeaders:
          - "*"
        AllowOrigins:
          - "*"

  ## Lambda function to generate signed URL
  UploadRequestFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: getSignedURL/
      Handler: app.handler
      Runtime: nodejs16.x
      MemorySize: 128
      Timeout: 5
      Environment:
        Variables:
          UploadBucket: !Ref S3UploadBucket
      Policies:
        - S3CrudPolicy:
            BucketName: !Ref S3UploadBucket
      Events:
        UploadAssetAPI:
          Type: HttpApi
          Properties:
            Path: /uploads
            Method: get
            ApiId: !Ref MyApi

  ## S3 bucket for uploads
  S3UploadBucket:
    Type: AWS::S3::Bucket
    Properties:
      CorsConfiguration:
        CorsRules:
          - AllowedHeaders:
              - "*"
            AllowedMethods:
              - GET
              - PUT
              - POST
              - DELETE
              - HEAD
            AllowedOrigins:
              - "*"

Outputs:
  APIendpoint:
    Description: "HTTP API endpoint URL"
    Value: !Sub "https://${MyApi}.execute-api.${AWS::Region}.amazonaws.com"
  S3UploadBucketName:
    Description: "S3 bucket for application uploads"
    Value: !Ref S3UploadBucket
```
## Lambda Function Code
The Lambda function generates a signed URL, which allows users to securely upload files to the S3 bucket.

## Lambda Function (app.js)
```javascript
'use strict'

const AWS = require('aws-sdk')
AWS.config.update({ region: process.env.AWS_REGION })
const s3 = new AWS.S3()

// URL expiration time in seconds
const URL_EXPIRATION_SECONDS = 300

// Main Lambda handler
exports.handler = async (event) => {
  return await getUploadURL(event)
}

const getUploadURL = async function(event) {
  const randomID = parseInt(Math.random() * 10000000)
  const Key = `${randomID}.jpg`

  // Parameters for generating signed URL
  const s3Params = {
    Bucket: process.env.UploadBucket,
    Key,
    Expires: URL_EXPIRATION_SECONDS,
    ContentType: 'image/jpeg',
  }

  const uploadURL = await s3.getSignedUrlPromise('putObject', s3Params)
  return JSON.stringify({
    uploadURL: uploadURL,
    Key
  })
}
```
## Front-End Code
The front-end interface allows users to select and upload images using the generated presigned URL.

```HTML + Vue.js + Axios
html
<!DOCTYPE html>
<html>
  <head>
    <title>Upload file to S3</title>
    <script src="https://unpkg.com/vue@1.0.28/dist/vue.js"></script>
    <script src="https://unpkg.com/axios@0.2.1/dist/axios.min.js"></script>
  </head>
  <body>
    <div id="app">
      <h1>S3 Uploader Test</h1>
      <div v-if="!image">
        <h2>Select an image</h2>
        <input type="file" @change="onFileChange" accept="image/jpeg">
      </div>
      <div v-else>
        <img :src="image" />
        <button v-if="!uploadURL" @click="removeImage">Remove image</button>
        <button v-if="!uploadURL" @click="uploadImage">Upload image</button>
      </div>
      <h2 v-if="uploadURL">Success! Image uploaded to bucket.</h2>
    </div>

    <script>
      const MAX_IMAGE_SIZE = 1000000
      const API_ENDPOINT = "https://n0wk4a7gk8.execute-api.us-east-1.amazonaws.com/uploads"

      new Vue({
        el: "#app",
        data: {
          image: '',
          uploadURL: ''
        },
        methods: {
          onFileChange (e) {
            let files = e.target.files || e.dataTransfer.files
            if (!files.length) return
            this.createImage(files[0])
          },
          createImage (file) {
            let reader = new FileReader()
            reader.onload = (e) => {
              if (!e.target.result.includes('data:image/jpeg')) {
                return alert('Wrong file type - JPG only.')
              }
              if (e.target.result.length > MAX_IMAGE_SIZE) {
                return alert('Image is too large.')
              }
              this.image = e.target.result
            }
            reader.readAsDataURL(file)
          },
          removeImage: function (e) {
            this.image = ''
          },
          uploadImage: async function (e) {
            const response = await axios.get(API_ENDPOINT)
            let binary = atob(this.image.split(',')[1])
            let array = []
            for (var i = 0; i < binary.length; i++) {
              array.push(binary.charCodeAt(i))
            }
            let blobData = new Blob([new Uint8Array(array)], { type: 'image/jpeg' })
            await fetch(response.data.uploadURL, {
              method: 'PUT',
              body: blobData
            })
            this.uploadURL = response.data.uploadURL.split('?')[0]
          }
        }
      })
    </script>

    <style>
      body {
        background: #20262E;
        padding: 20px;
        font-family: sans-serif;
      }
      #app {
        background: #fff;
        border-radius: 4px;
        padding: 20px;
        text-align: center;
      }
      img {
        width: 30%;
        margin: auto;
        display: block;
        margin-bottom: 10px;
      }
    </style>
  </body>
</html>
```
## Setup & Deployment
### Prerequisites
AWS Account: You need an AWS account with appropriate permissions.
AWS SAM CLI: Install the AWS SAM CLI to build and deploy the application.
Node.js: Ensure Node.js is installed for local development.
## Steps to Deploy
### Build the application:
```bash
sam build
```
### Deploy the application:
```bash
sam deploy --guided
```
View the application: After deployment, visit the API Gateway endpoint to interact with the application and upload images.
