# APIs-Lambda-and-S3

Here’s a detailed breakdown of how to implement the required API service using AWS Lambda, API Gateway, and S3:

### Step 1: Set Up AWS Resources

#### 1.1. Create an S3 Bucket
   - Log in to your AWS account, go to **S3**, and click **Create Bucket**.
   - Give your bucket a unique name, like `my-json-storage-bucket`.
   - Set the bucket to be **public** if necessary for this exercise (though in real-world cases, it’s better to use secure access permissions).
   - Under **Permissions**, adjust access settings so the bucket allows public read and write access or, alternatively, configure the appropriate IAM roles (more secure).
   - Note the **bucket name** as it will be required for your Lambda functions.

#### 1.2. Set Up API Gateway
   - Go to **API Gateway** in AWS.
   - Click on **Create API** and select **HTTP API** for a simpler setup.
   - Name your API, e.g., `JSONStorageAPI`.
   - Configure **routes**:
      - **POST /store**: To handle data storage in S3.
      - **GET /retrieve**: To handle data retrieval from S3.
   - After creating routes, set up **integrations** to connect these routes with the Lambda functions you’ll create next.

### Step 2: Implement Lambda Functions

#### 2.1. Lambda Function for the POST Endpoint (`storeData`)
   - Go to **AWS Lambda** and click **Create Function**.
   - Name your function, e.g., `storeData`.
   - Use **Node.js** (recommended) or Python as the runtime.
   - Under **Permissions**, choose an existing role or create a new role with permissions for API Gateway and S3 access.
   - In the Lambda function code editor, implement the POST handler to:
     - Receive the JSON payload.
     - Generate a unique filename (e.g., `UUID.json`).
     - Store the JSON data in the S3 bucket.

##### Sample Code (Node.js):
```javascript
const AWS = require('aws-sdk');
const s3 = new AWS.S3();
const { v4: uuidv4 } = require('uuid');

exports.handler = async (event) => {
    try {
        const data = JSON.parse(event.body); // parse the JSON payload
        const filename = `${uuidv4()}.json`; // generate a unique filename

        const params = {
            Bucket: 'my-json-storage-bucket', // replace with your bucket name
            Key: filename,
            Body: JSON.stringify(data),
            ContentType: 'application/json'
        };

        const result = await s3.putObject(params).promise();
        
        // Return ETag and S3 URL to the user
        return {
            statusCode: 200,
            body: JSON.stringify({
                e_tag: result.ETag,
                url: `https://${params.Bucket}.s3.amazonaws.com/${filename}`
            })
        };
    } catch (error) {
        return {
            statusCode: 500,
            body: JSON.stringify({ error: 'Could not store data', details: error.message })
        };
    }
};
```

   - Replace `my-json-storage-bucket` with the actual bucket name you created.
   - Save the function and test it with a sample event containing JSON data in the `body`.

#### 2.2. Lambda Function for the GET Endpoint (`retrieveData`)
   - Create another Lambda function named `retrieveData`.
   - Ensure the function has permissions to read from the S3 bucket.
   - This function will fetch all JSON files from the bucket, compile the contents, and return them as a single JSON array.

##### Sample Code (Node.js):
```javascript
const AWS = require('aws-sdk');
const s3 = new AWS.S3();

exports.handler = async () => {
    try {
        const listParams = {
            Bucket: 'my-json-storage-bucket' // replace with your bucket name
        };
        const listedObjects = await s3.listObjectsV2(listParams).promise();
        
        const filePromises = listedObjects.Contents.map(async (file) => {
            const fileData = await s3.getObject({ Bucket: listParams.Bucket, Key: file.Key }).promise();
            return JSON.parse(fileData.Body.toString());
        });
        
        const files = await Promise.all(filePromises);

        return {
            statusCode: 200,
            body: JSON.stringify(files)
        };
    } catch (error) {
        return {
            statusCode: 500,
            body: JSON.stringify({ error: 'Could not retrieve data', details: error.message })
        };
    }
};
```

   - Test this function to ensure it retrieves and compiles all JSON objects from the S3 bucket.

### Step 3: Connect Lambda Functions to API Gateway

1. In **API Gateway**, open the **Routes** tab.
2. Select the **POST /store** route:
   - Attach the **storeData** Lambda function as the backend integration.
3. Select the **GET /retrieve** route:
   - Attach the **retrieveData** Lambda function.
4. Deploy the API and note down the endpoint URLs for testing.

### Step 4: Test the API Endpoints

#### 4.1 Testing the POST Endpoint
   - Use a tool like **Postman** or **curl** to send a POST request.
   - Sample request:
     ```json
     {
       "name": "John",
       "age": 30
     }
     ```
   - You should receive a response with the S3 `e_tag` and `url`.

#### 4.2 Testing the GET Endpoint
   - Use **Postman** or **curl** to send a GET request.
   - The response should be a JSON array of all the objects stored in your S3 bucket.

### Step 5: Additional Considerations

1. **Error Handling**: Ensure that Lambda functions handle:
   - Invalid JSON input (POST endpoint).
   - S3 permissions errors.
   - Edge cases, such as empty bucket retrieval.

2. **Permissions**:
   - Ensure your Lambda functions have **S3 read and write permissions**.
   - Consider **IAM roles and policies** to restrict access based on specific needs.

3. **Security**:
   - Restrict access to the API Gateway with an **API Key** or **IAM** permissions if this is a production service.
   - Set proper **CORS** settings if this API is consumed by web clients.

This setup will give you a basic web service with storage and retrieval capabilities using AWS Lambda, API Gateway, and S3.
