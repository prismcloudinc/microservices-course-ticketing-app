# Ticketing Application using Microservices Architecture

This is a Ticketing Application built using Microservices Architecture. The project is part of the [Microservices with Node.js and React](https://www.udemy.com/course/microservices-with-node-js-and-react/) course by Stephen Grider on Udemy. By the time I started the course, the course was using older versions of the libraries and tools. I decided to update the project to use the latest versions of the libraries and tools.

Changes worth mentioning:

- Replaced `npm` with `pnpm` package manager.
- Added `Turbo` to manage the project as a monorepo. It is helpful to manage dependencies and test the project as a whole.
- Updated the libraries and tools to the latest versions.
  - Built on Node.js v20, used React v18 and Next.js v14 with App Router.
- Up-to-date approach for running tests in GitHub Actions using pnpm cache and manually cached mongodb binaries.
- Health check endpoints for each service and client for Kubernetes liveness checks to ensure the services are running properly.

## Table of Contents

- [Technologies Used](#technologies-used)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
- [Testing](#testing)
- [Manual Testing with Postman](#manual-testing-with-postman)
- [Managing Common Module](#managing-common-module)
- [Architecture](#architecture)

## Technologies Used

- Turbo with pnpm
- Express.js (v4.x)
- MongoDB & Mongoose
- Jest & Supertest
- React.js (v18.x)
- Next.js (v14.x) with App Router
- TypeScript (v5.x)
- Tailwindcss
- Docker
- Kubernetes
- NATS Streaming Server
- Skaffold

## Getting Started

### Prerequisites

- Node.js (>=20)
  - Using [nvm](https://github.com/nvm-sh/nvm?tab=readme-ov-file#installing-and-updating) makes thing super easy.
  - Although, nvm doesn't suggest using [homebrew](https://brew.sh/), it is possible:
    ```bash
    brew install nvm
    nvm install 20
    ```
- Pnpm (>=9)
  - Again brew installation is the easiest, getting never versions is much easier compared to globally installed npm package.
    ```bash
    brew install pnpm
    ```
- Docker & Kubernetes
  - [Docker Desktop](https://www.docker.com/products/docker-desktop/) is the easiest to install both
  - Enable Kubernetes in Docker Desktop settings
- Ingress Nginx Controller & Skaffold
  - Install [Ingress Nginx](https://kubernetes.github.io/ingress-nginx/deploy/#quick-start:~:text=manifest%2C%20you%20can-,run%20the%20following,-command%20instead%3A)
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
    ```
  - Install [Skaffold](https://skaffold.dev/docs/install/#homebrew)
    ```bash
    brew install skaffold
    ```
- Configure hosts file
  - Add the following line to your `/etc/hosts` file:
    ```bash
    127.0.0.1 ticketing.dev
    ```
- [Stripe](https://docs.stripe.com/test-mode) account with test access

### Installation

**Clone the repository:**

```bash
git clone https://github.com/berkinanik/microservices-course-ticketing-app.git ticketing-app
cd ticketing-app
```

**Clone the [common module](https://github.com/berkinanik/microservices-course-common-module):**

```bash
cd packages/common
git clone https://github.com/berkinanik/microservices-course-common-module.git .
cd ../..
```

**Use Node.js v20**

`.nvmrc` file specifies the Node.js version to use. To switch to the specified version, run:

```bash
nvm use
```

**Install dependencies:**

```bash
pnpm install
pnpm lockfile
```

_`lockfile` command uses `@pnpm/make-dedicated-lockfile` to create lockfile for each service by removing package links created by turbo repo. Which are used by Dockerfile of each project while building their images._

**Set up environment variables**

There are couple environment variables that are needed to be set in order to run the application. Those must be added to kubectl secrets and mounted to pods as environment variables.

_Replace values between `<` and `>` with your own values._

- `JWT_KEY`: Secret key for JWT token generation.

```bash
kubectl create secret generic jwt-secret --from-literal=JWT_KEY=<your_secret_key>
```

- `STRIPE_KEY`: Stripe secret key for payment processing.

```bash
kubectl create secret generic stripe-secret --from-literal=STRIPE_KEY=<your_stripe_secret_key>
```

- `STRIPE_CLIENT_KEY`: Stripe public key for payment processing.

```bash
kubectl create secret generic stripe-client-secret --from-literal=STRIPE_CLIENT_KEY=<your_stripe_client_key>
```

**Start the application:**

In the root (where the `skaffold.yaml` file is located), run:

```bash
skaffold dev
```

**Access the application in your browser:**

```bash
https://ticketing.dev
```

_If chrome gives you a warning about the certificate, type in your keyboard while the page is open: `thisisunsafe`. After 2~3 seconds chrome will load the page._

## Testing

There are unit tests for each service. Both routes and functionalities around event listeners are tested. For testing routes `supertest` is used, and `mongodb-memory-server` is used for mongodb instance in jest environment. To run the tests, navigate to the service directory and run:
You can run tests for each service using pnpm workspaces from the root directory.

- To run a specific service's tests:
  ```bash
  pnpm test --filter <service_name>
  ```
  Example:
  ```bash
  pnpm test --filter auth
  ```
- To run all services' tests:
  ```bash
  pnpm test
  ```
- To run all services' tests in watch mode:
  ```bash
  pnpm test:watch
  ```
- Or run individual service's tests in watch mode:
  ```bash
  pnpm test:watch --filter <service_name>
  ```
  Example:
  ```bash
  pnpm test:watch --filter auth
  ```

## Manual Testing with Postman

For testing the services manually, you can use the Postman collection provided in the `postman` directory. The collection contains requests for all the services in the application. Environment variables are also provided with the collection to set the base URL for the requests.

## Managing Common Module

The `common` module is a shared module that contains common code used by multiple services. It is published to npm and used by other services as a dependency. Currently, the module is published to npm with the `@b.anik/ticketing-common` package name.

If you want to make changes to the `common` module and test it in the services, you can change the name of the package in the `common` module's `package.json` file to something else. Replace all occurrences of the package name (`@b.anik/ticketing-common`) in the services with the new package name.

Then run following command to publish the `common` module under your npm account

```bash
pnpm pub:common
```

You can update the common package version in all services by running the following command in the root directory:

```bash
pnpm update:common
```

_Make sure you also changed the package name in the package.json update:common script in the root._

## Architecture

The Ticketing Application follows a microservices architecture, where different components of the application are developed and deployed as separate services. The services communicate with each other using an event-driven approach.

The main services in the application are:

- **Auth Service**: Responsible for user authentication and authorization.
- **Tickets Service**: Handles ticket creation, management, and availability.
- **Orders Service**: Manages the creation and management of orders.
- **Payments Service**: Handles payment processing for orders.
- **Expiration Service**: Listens for order creation events and cancels orders after a certain period of time.
- **Client**: The frontend application built using Next.js.

## License

This project is licensed under the [MIT License](LICENSE).
