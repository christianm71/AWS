# API Gateway with Lambda function integration


## Architecture Overview

```
Client → API Gateway → Lambda Function → Backend Services
         (HTTP API)    (Processing)     (DB, S3, etc.)
```


## Lambda Function (Python)

```python
import json
import boto3
from datetime import datetime

def lambda_handler(event, context):
    """
    Process API Gateway requests
    event contains request data from API Gateway
    """
    
    # Parse HTTP method and route
    http_method = event.get('httpMethod', 'GET')
    path = event.get('path', '/')
    
    try:
        # Process based on HTTP method and path
        if http_method == 'GET' and path == '/users':
            return get_users()
        elif http_method == 'POST' and path == '/users':
            return create_user(event)
        elif http_method == 'GET' and path.startswith('/users/'):
            user_id = path.split('/')[-1]
            return get_user(user_id)
        else:
            return error_response(404, "Route not found")
            
    except Exception as e:
        return error_response(500, f"Internal server error: {str(e)}")

def get_users():
    """Get all users"""
    # Simulate database query
    users = [
        {"id": 1, "name": "John Doe", "email": "john@example.com"},
        {"id": 2, "name": "Jane Smith", "email": "jane@example.com"}
    ]
    
    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*"
        },
        "body": json.dumps({
            "users": users,
            "timestamp": datetime.now().isoformat()
        })
    }

def create_user(event):
    """Create a new user"""
    # Parse request body
    body = json.loads(event.get('body', '{}'))
    
    # Validate required fields
    required_fields = ['name', 'email']
    for field in required_fields:
        if field not in body:
            return error_response(400, f"Missing required field: {field}")
    
    # Process user creation (simulated)
    new_user = {
        "id": 3,  # This would come from database
        "name": body['name'],
        "email": body['email'],
        "created_at": datetime.now().isoformat()
    }
    
    return {
        "statusCode": 201,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*"
        },
        "body": json.dumps({
            "message": "User created successfully",
            "user": new_user
        })
    }

def get_user(user_id):
    """Get specific user by ID"""
    # Simulate database lookup
    users = {
        "1": {"id": 1, "name": "John Doe", "email": "john@example.com"},
        "2": {"id": 2, "name": "Jane Smith", "email": "jane@example.com"}
    }
    
    user = users.get(user_id)
    if not user:
        return error_response(404, f"User {user_id} not found")
    
    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*"
        },
        "body": json.dumps({"user": user})
    }

def error_response(status_code, message):
    """Standard error response"""
    return {
        "statusCode": status_code,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*"
        },
        "body": json.dumps({
            "error": message,
            "timestamp": datetime.now().isoformat()
        })
    }
```


## API Gateway Configuration

### API Gateway Configuration

```yaml
openapi: 3.0.0
info:
  title: User Management API
  version: 1.0.0
paths:
  /users:
    get:
      summary: Get all users
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  users:
                    type: array
                    items:
                      type: object
                      properties:
                        id:
                          type: integer
                        name:
                          type: string
                        email:
                          type: string
    post:
      summary: Create a new user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - name
                - email
              properties:
                name:
                  type: string
                email:
                  type: string
      responses:
        '201':
          description: User created
  /users/{userId}:
    get:
      summary: Get user by ID
      parameters:
        - name: userId
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Successful response
        '404':
          description: User not found
```

### Infrastructure as Code (AWS SAM)

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  # Lambda Function
  UserApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: app.lambda_handler
      Runtime: python3.9
      Timeout: 30
      MemorySize: 256
      Environment:
        Variables:
          LOG_LEVEL: INFO
      Events:
        GetUsers:
          Type: Api
          Properties:
            Path: /users
            Method: get
        CreateUser:
          Type: Api
          Properties:
            Path: /users
            Method: post
        GetUser:
          Type: Api
          Properties:
            Path: /users/{userId}
            Method: get

  # API Gateway
  ApiGateway:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      Cors:
        AllowMethods: "'GET,POST,PUT,DELETE,OPTIONS'"
        AllowHeaders: "'Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token'"
        AllowOrigin: "'*'"

Outputs:
  ApiUrl:
    Description: "API Gateway endpoint URL"
    Value: !Sub "https://${ApiGateway}.execute-api.${AWS::Region}.amazonaws.com/prod/"
```

