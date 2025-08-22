# AWS Setup and Project Structure Guide

## Phase 1, Step 3: AWS Setup

### 3.1 Amazon Cognito User Pool Configuration

#### Prerequisites
- AWS CLI installed and configured with appropriate credentials
- AWS SAM CLI installed for serverless deployment
- Appropriate AWS account with necessary permissions

#### Step-by-Step Cognito Setup

1. **Create Cognito User Pool**
   ```bash
   # Using AWS CLI
   aws cognito-idp create-user-pool \
     --pool-name "SurveyAppUserPool" \
     --policies "PasswordPolicy={MinimumLength=8,RequireUppercase=true,RequireLowercase=true,RequireNumbers=true,RequireSymbols=true}" \
     --auto-verified-attributes email \
     --username-attributes email
   ```

2. **Create User Pool Client**
   ```bash
   # Replace USER_POOL_ID with the ID from step 1
   aws cognito-idp create-user-pool-client \
     --user-pool-id "USER_POOL_ID" \
     --client-name "SurveyAppClient" \
     --no-generate-secret \
     --explicit-auth-flows "ALLOW_USER_PASSWORD_AUTH" "ALLOW_REFRESH_TOKEN_AUTH" "ALLOW_USER_SRP_AUTH"
   ```

3. **Configure App Integration**
   - Set callback URLs: `http://localhost:3000/auth/callback`
   - Set sign-out URLs: `http://localhost:3000`
   - Enable OAuth flows: Authorization code grant, Implicit grant

#### Environment Variables
Create `.env.local` file:
```env
REACT_APP_COGNITO_USER_POOL_ID=your_user_pool_id
REACT_APP_COGNITO_CLIENT_ID=your_client_id
REACT_APP_COGNITO_REGION=your_aws_region
```

### 3.2 DynamoDB Table Setup

#### Survey Table
```bash
aws dynamodb create-table \
  --table-name "SurveyTable" \
  --attribute-definitions \
    AttributeName=SurveyID,AttributeType=S \
    AttributeName=AdminUserID,AttributeType=S \
  --key-schema \
    AttributeName=SurveyID,KeyType=HASH \
  --global-secondary-indexes \
    IndexName=AdminUserIDIndex,KeySchema=[{AttributeName=AdminUserID,KeyType=HASH}],Projection={ProjectionType=ALL} \
  --billing-mode PAY_PER_REQUEST
```

#### Questions Table
```bash
aws dynamodb create-table \
  --table-name "QuestionsTable" \
  --attribute-definitions \
    AttributeName=QuestionID,AttributeType=S \
    AttributeName=SurveyID,AttributeType=S \
  --key-schema \
    AttributeName=QuestionID,KeyType=HASH \
  --global-secondary-indexes \
    IndexName=SurveyIDIndex,KeySchema=[{AttributeName=SurveyID,KeyType=HASH}],Projection={ProjectionType=ALL} \
  --billing-mode PAY_PER_REQUEST
```

#### Options Table
```bash
aws dynamodb create-table \
  --table-name "OptionsTable" \
  --attribute-definitions \
    AttributeName=OptionID,AttributeType=S \
    AttributeName=QuestionID,AttributeType=S \
  --key-schema \
    AttributeName=OptionID,KeyType=HASH \
  --global-secondary-indexes \
    IndexName=QuestionIDIndex,KeySchema=[{AttributeName=QuestionID,KeyType=HASH}],Projection={ProjectionType=ALL} \
  --billing-mode PAY_PER_REQUEST
```

#### Responses Table
```bash
aws dynamodb create-table \
  --table-name "ResponsesTable" \
  --attribute-definitions \
    AttributeName=ResponseID,AttributeType=S \
    AttributeName=SurveyID,AttributeType=S \
  --key-schema \
    AttributeName=ResponseID,KeyType=HASH \
  --global-secondary-indexes \
    IndexName=SurveyIDIndex,KeySchema=[{AttributeName=SurveyID,KeyType=HASH}],Projection={ProjectionType=ALL} \
  --billing-mode PAY_PER_REQUEST
```

### 3.3 Lambda Functions Setup

#### Create SAM Template
Create `template.yaml`:
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Survey App Lambda Functions

Globals:
  Function:
    Timeout: 30
    Runtime: nodejs18.x
    Environment:
      Variables:
        SURVEY_TABLE: !Ref SurveyTable
        QUESTIONS_TABLE: !Ref QuestionsTable
        OPTIONS_TABLE: !Ref OptionsTable
        RESPONSES_TABLE: !Ref ResponsesTable

Resources:
  # Lambda Functions
  CreateSurveyFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: lambda/create-survey/
      Handler: index.handler
      Events:
        Api:
          Type: Api
          Properties:
            Path: /surveys
            Method: post

  GetSurveysFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: lambda/get-surveys/
      Handler: index.handler
      Events:
        Api:
          Type: Api
          Properties:
            Path: /surveys
            Method: get

  # DynamoDB Tables (if not created separately)
  SurveyTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: SurveyTable
      AttributeDefinitions:
        - AttributeName: SurveyID
          AttributeType: S
      KeySchema:
        - AttributeName: SurveyID
          KeyType: HASH
      BillingMode: PAY_PER_REQUEST

  QuestionsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: QuestionsTable
      AttributeDefinitions:
        - AttributeName: QuestionID
          AttributeType: S
      KeySchema:
        - AttributeName: QuestionID
          KeyType: HASH
      BillingMode: PAY_PER_REQUEST

  OptionsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: OptionsTable
      AttributeDefinitions:
        - AttributeName: OptionID
          AttributeType: S
      KeySchema:
        - AttributeName: OptionID
          KeyType: HASH
      BillingMode: PAY_PER_REQUEST

  ResponsesTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: ResponsesTable
      AttributeDefinitions:
        - AttributeName: ResponseID
          AttributeType: S
      KeySchema:
        - AttributeName: ResponseID
          KeyType: HASH
      BillingMode: PAY_PER_REQUEST

Outputs:
  ApiGatewayApi:
    Description: API Gateway endpoint URL
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/"
  CreateSurveyFunction:
    Description: Create Survey Lambda Function ARN
    Value: !GetAtt CreateSurveyFunction.Arn
  GetSurveysFunction:
    Description: Get Surveys Lambda Function ARN
    Value: !GetAtt GetSurveysFunction.Arn
```

#### Deploy with SAM
```bash
# Build the application
sam build

# Deploy to AWS
sam deploy --guided
```

### 3.4 IAM Roles and Policies

#### Lambda Execution Role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query",
        "dynamodb:Scan"
      ],
      "Resource": [
        "arn:aws:dynamodb:*:*:table/SurveyTable",
        "arn:aws:dynamodb:*:*:table/QuestionsTable",
        "arn:aws:dynamodb:*:*:table/OptionsTable",
        "arn:aws:dynamodb:*:*:table/ResponsesTable"
      ]
    }
  ]
}
```

## Phase 1, Step 4: Project Structure Organization

### 4.1 Frontend Project Structure

#### Directory Organization
```
survey-app/
├── public/                 # Static assets
├── src/
│   ├── components/         # Reusable UI components
│   │   ├── ui/            # shadcn/ui components
│   │   ├── forms/         # Form-related components
│   │   ├── layout/        # Layout components
│   │   └── common/        # Common utility components
│   ├── pages/             # Application pages
│   │   ├── auth/          # Authentication pages
│   │   ├── dashboard/     # Main dashboard
│   │   ├── survey/        # Survey creation/editing
│   │   └── public/        # Public survey pages
│   ├── services/          # API and external services
│   │   ├── api/           # API client functions
│   │   ├── cognito/       # Cognito authentication
│   │   └── storage/       # Local storage utilities
│   ├── types/             # TypeScript type definitions
│   │   ├── survey.ts      # Survey-related types
│   │   ├── user.ts        # User-related types
│   │   └── api.ts         # API response types
│   ├── hooks/             # Custom React hooks
│   │   ├── useAuth.ts     # Authentication hook
│   │   ├── useSurveys.ts  # Survey management hook
│   │   └── useForm.ts     # Form handling hook
│   ├── contexts/          # React context providers
│   │   ├── AuthContext.tsx # Authentication context
│   │   └── SurveyContext.tsx # Survey state context
│   ├── lib/               # Utility functions
│   │   ├── utils.ts       # General utilities
│   │   ├── validation.ts  # Validation schemas
│   │   └── constants.ts   # Application constants
│   ├── styles/            # Additional styles
│   ├── App.tsx            # Main application component
│   ├── index.tsx          # Application entry point
│   └── index.css          # Global styles
├── package.json            # Dependencies and scripts
├── tsconfig.json          # TypeScript configuration
├── tailwind.config.js     # Tailwind CSS configuration
├── components.json        # shadcn/ui configuration
└── README.md              # Project documentation
```

### 4.2 Component Architecture

#### UI Component Hierarchy
```
components/
├── ui/                    # shadcn/ui base components
│   ├── button.tsx        # Button component
│   ├── input.tsx         # Input component
│   ├── card.tsx          # Card component
│   ├── dialog.tsx        # Modal dialog
│   └── form.tsx          # Form components
├── forms/                 # Survey-specific forms
│   ├── SurveyForm.tsx    # Main survey creation form
│   ├── QuestionForm.tsx  # Question creation form
│   └── ResponseForm.tsx  # Survey response form
├── layout/                # Layout components
│   ├── Header.tsx        # Application header
│   ├── Sidebar.tsx       # Navigation sidebar
│   ├── Footer.tsx        # Application footer
│   └── Layout.tsx        # Main layout wrapper
└── common/                # Common components
    ├── Loading.tsx        # Loading spinner
    ├── ErrorBoundary.tsx  # Error handling
    └── ConfirmDialog.tsx  # Confirmation dialogs
```

### 4.3 Type Definitions

#### Core Types (`src/types/survey.ts`)
```typescript
export interface Survey {
  surveyId: string;
  name: string;
  description?: string;
  createdAt: string;
  updatedAt: string;
  adminUserId: string;
  status: 'draft' | 'active' | 'inactive';
  shareableLink: string;
  questions: Question[];
}

export interface Question {
  questionId: string;
  surveyId: string;
  text: string;
  type: 'multiple_choice' | 'single_choice';
  required: boolean;
  order: number;
  options: Option[];
}

export interface Option {
  optionId: string;
  questionId: string;
  text: string;
  order: number;
}

export interface SurveyResponse {
  responseId: string;
  surveyId: string;
  respondentId: string;
  submittedAt: string;
  answers: Answer[];
}

export interface Answer {
  questionId: string;
  selectedOptions: string[];
}
```

#### User Types (`src/types/user.ts`)
```typescript
export interface User {
  userId: string;
  email: string;
  username: string;
  attributes: {
    email_verified: boolean;
    sub: string;
  };
}

export interface AuthState {
  isAuthenticated: boolean;
  user: User | null;
  tokens: {
    accessToken: string;
    refreshToken: string;
    idToken: string;
  } | null;
  loading: boolean;
}
```

### 4.4 Service Layer Structure

#### API Service (`src/services/api/client.ts`)
```typescript
import axios from 'axios';

const API_BASE_URL = process.env.REACT_APP_API_BASE_URL || 'https://your-api-gateway-url.amazonaws.com/Prod';

export const apiClient = axios.create({
  baseURL: API_BASE_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor for authentication
apiClient.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('accessToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor for error handling
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      // Handle token refresh or redirect to login
      localStorage.removeItem('accessToken');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

#### Survey Service (`src/services/api/surveys.ts`)
```typescript
import { apiClient } from './client';
import { Survey, CreateSurveyRequest } from '../../types/survey';

export const surveyService = {
  // Get all surveys for the authenticated user
  async getSurveys(): Promise<Survey[]> {
    const response = await apiClient.get('/surveys');
    return response.data;
  },

  // Create a new survey
  async createSurvey(surveyData: CreateSurveyRequest): Promise<Survey> {
    const response = await apiClient.post('/surveys', surveyData);
    return response.data;
  },

  // Get a specific survey by ID
  async getSurvey(surveyId: string): Promise<Survey> {
    const response = await apiClient.get(`/surveys/${surveyId}`);
    return response.data;
  },

  // Update an existing survey
  async updateSurvey(surveyId: string, surveyData: Partial<Survey>): Promise<Survey> {
    const response = await apiClient.put(`/surveys/${surveyId}`, surveyData);
    return response.data;
  },

  // Delete a survey
  async deleteSurvey(surveyId: string): Promise<void> {
    await apiClient.delete(`/surveys/${surveyId}`);
  },

  // Submit survey response
  async submitResponse(surveyId: string, responseData: any): Promise<void> {
    await apiClient.post(`/surveys/${surveyId}/responses`, responseData);
  },
};
```

### 4.5 Context Providers

#### Authentication Context (`src/contexts/AuthContext.tsx`)
```typescript
import React, { createContext, useContext, useReducer, useEffect } from 'react';
import { AuthState, User } from '../types/user';
import { cognitoService } from '../services/cognito';

interface AuthContextType extends AuthState {
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  refreshToken: () => Promise<void>;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

const initialState: AuthState = {
  isAuthenticated: false,
  user: null,
  tokens: null,
  loading: true,
};

type AuthAction =
  | { type: 'SET_LOADING'; payload: boolean }
  | { type: 'SET_USER'; payload: User }
  | { type: 'SET_TOKENS'; payload: any }
  | { type: 'LOGOUT' }
  | { type: 'SET_AUTHENTICATED'; payload: boolean };

function authReducer(state: AuthState, action: AuthAction): AuthState {
  switch (action.type) {
    case 'SET_LOADING':
      return { ...state, loading: action.payload };
    case 'SET_USER':
      return { ...state, user: action.payload, isAuthenticated: true };
    case 'SET_TOKENS':
      return { ...state, tokens: action.payload };
    case 'LOGOUT':
      return { ...initialState, loading: false };
    case 'SET_AUTHENTICATED':
      return { ...state, isAuthenticated: action.payload };
    default:
      return state;
  }
}

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(authReducer, initialState);

  useEffect(() => {
    // Check for existing session on app load
    checkAuthStatus();
  }, []);

  const checkAuthStatus = async () => {
    try {
      const user = await cognitoService.getCurrentUser();
      if (user) {
        dispatch({ type: 'SET_USER', payload: user });
        dispatch({ type: 'SET_AUTHENTICATED', payload: true });
      }
    } catch (error) {
      console.error('Auth check failed:', error);
    } finally {
      dispatch({ type: 'SET_LOADING', payload: false });
    }
  };

  const login = async (email: string, password: string) => {
    dispatch({ type: 'SET_LOADING', payload: true });
    try {
      const { user, tokens } = await cognitoService.signIn(email, password);
      dispatch({ type: 'SET_USER', payload: user });
      dispatch({ type: 'SET_TOKENS', payload: tokens });
      dispatch({ type: 'SET_AUTHENTICATED', payload: true });
    } finally {
      dispatch({ type: 'SET_LOADING', payload: false });
    }
  };

  const logout = async () => {
    await cognitoService.signOut();
    dispatch({ type: 'LOGOUT' });
  };

  const refreshToken = async () => {
    try {
      const tokens = await cognitoService.refreshToken();
      dispatch({ type: 'SET_TOKENS', payload: tokens });
    } catch (error) {
      dispatch({ type: 'LOGOUT' });
    }
  };

  const value: AuthContextType = {
    ...state,
    login,
    logout,
    refreshToken,
  };

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
}
```

### 4.6 Environment Configuration

#### Environment Variables (`.env.local`)
```env
# AWS Configuration
REACT_APP_AWS_REGION=us-east-1
REACT_APP_COGNITO_USER_POOL_ID=us-east-1_xxxxxxxxx
REACT_APP_COGNITO_CLIENT_ID=xxxxxxxxxxxxxxxxxxxxxxxxxx

# API Configuration
REACT_APP_API_BASE_URL=https://your-api-gateway-url.amazonaws.com/Prod

# Application Configuration
REACT_APP_APP_NAME=Survey Creation App
REACT_APP_APP_VERSION=1.0.0
REACT_APP_ENVIRONMENT=development
```

### 4.7 Next Steps After Setup

1. **Install Additional Dependencies**
   ```bash
   npm install axios @aws-amplify/ui-react aws-amplify
   ```

2. **Configure AWS Amplify** (if using Amplify)
   ```typescript
   // src/aws-exports.js
   const awsConfig = {
     Auth: {
       region: process.env.REACT_APP_AWS_REGION,
       userPoolId: process.env.REACT_APP_COGNITO_USER_POOL_ID,
       userPoolWebClientId: process.env.REACT_APP_COGNITO_CLIENT_ID,
     },
   };
   ```

3. **Test the Setup**
   - Run `npm start` to start the development server
   - Verify Tailwind CSS is working by adding Tailwind classes
   - Test shadcn/ui components by importing and using them
   - Verify the project structure is correctly organized

4. **Begin Phase 2: Authentication System**
   - Implement Cognito integration
   - Create login/logout components
   - Set up protected routes
   - Implement session management

This setup provides a solid foundation for building the survey application with proper separation of concerns, type safety, and scalable architecture.
