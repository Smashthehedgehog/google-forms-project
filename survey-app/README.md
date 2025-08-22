# Survey Creation Application

A full-stack survey creation and management application built with React, TypeScript, Tailwind CSS, and AWS services.

## Features

- **Authentication**: Secure admin login using Amazon Cognito
- **Survey Management**: Create, edit, delete, and manage surveys
- **Dynamic Question Builder**: Add unlimited questions with multiple question types
- **Public Survey Access**: Shareable links for survey respondents
- **Response Collection**: Store and analyze survey responses
- **Modern UI**: Built with Tailwind CSS and shadcn/ui components

## Tech Stack

- **Frontend**: React 18+ with TypeScript
- **Styling**: Tailwind CSS
- **UI Components**: shadcn/ui
- **Backend**: AWS Serverless (Lambda, API Gateway, DynamoDB)
- **Authentication**: Amazon Cognito
- **State Management**: React Context API

## Getting Started

1. Install dependencies: `npm install`
2. Start development server: `npm start`
3. Build for production: `npm run build`

## Project Structure

```
src/
├── components/     # Reusable UI components
├── pages/         # Application pages
├── services/      # API and external service integrations
├── types/         # TypeScript type definitions
├── hooks/         # Custom React hooks
├── contexts/      # React context providers
├── lib/           # Utility functions
└── App.tsx        # Main application component
```

## Development

This project follows a modular architecture with clear separation of concerns. Each component, service, and utility is documented and follows best practices for maintainability and scalability.
