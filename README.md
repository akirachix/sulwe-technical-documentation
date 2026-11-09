# sulwe-technical-documentation
Technical documentation for the sulwe project

This is the technical documentation for Umucyo product, a digital platform that helps tea cooperatives manage tea collection, farmer records and farmer payments.

Umucyo is used by three kinds of people. Cooperative administrators manage the cooperative from the admin dashboard. Collection-point agents register farmers and record tea deliveries in the agent app. Farmers, who often have only basic phones, receive SMS confirmations and use USSD to check their reports.

The system is made of three parts. A FastAPI backend with a PostgreSQL database holds all the logic and data. A Next.js admin dashboard serves the administrators, and a Next.js agent app serves the agents. The backend also connects to outside services: M-Pesa Daraja for payments, email for login codes and invitations, SMS Leopard for farmer messages, and Africa's Talking for USSD.

## What the documentation covers

The documentation explains how each feature works, how the parts of the system fit together, and how to set up, test and run the project. It describes these features:

- User registration and account management, including the controlled registration of administrators, agents and farmers, and secure login.
- Collection points, where farmers deliver their tea and agents work.
- Weighing device management, which keeps a registry of the scales used to weigh deliveries.
- Produce records, which capture each farmer's tea delivery and calculate the amount owed.
- SMS delivery notifications, which tell farmers when their deliveries are recorded.
- Tea pricing, which manages the price paid per kilogram over time.
- Tasks management, which lets administrators assign and track work for agents.
- Payments, which pay farmers every month through M-Pesa.
- USSD reporting, which gives farmers access to their information from basic phones in English and Kinyarwanda.

It also covers the technology used, the project's code conventions, the main database tables, the API endpoints, the configuration settings, local setup, testing, deployment and security.

## Who it is for

The documentation is written for developers joining or working on the project, for reviewers checking how features behave, and for anyone who needs to set up and run Umucyo.

Author: Sulwe Group