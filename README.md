

## 2. Improving Ruuter Documentation.

### Define which sections of documentation should be updated.

Main Readme File
- Initial README file does not hold any information on what Ruuter is or does, it's made from generic information. For users who already work with it, does not hold much value, but for new developers to join and start working with it, its quite unclear on its purpose.
- The initial structure is there, but its missing an entire Intro block what would be more expressive about ruuter.
- Guide it self have solid information but structure might be updated, adding all possible elements to bottom of it and more general over view to the guide file.

### How to enhance or restructure documentation for newer developers.

- Main Readme File holds no information of Ruuter overview to give more understanding
- Configuration docs should be more explicit on configuration parameters.
- Guide file should hold more General examples to outline Ruuters functionality and purpose
- Would be Good to include big example file within main Readme or Guide file to emphasis Ruuter

### How to document use and validation of inputs

- Security aspect is fully missed from documents.
- As currently GUIDE holds use related files, for consistency we can keep all related information in this file.
- As SECONDARY option we can make an input validation document and Link location from inside guide
- As SECONDARY option for security aspect we can put extra document covering security aspect of using ruuter and link from inside guide

## 3. Improving Resql Documentation

### How to improve existing Swagger UI API Documentation

- Update dependency to include a proper library to display swagger page
- Fix Controllers to either have a separate swagger controller, or ensure that current controller is not blocking access to the page by adding something like  /api to root of controller.
- Current version of code base does not have much configurations
- configure all files via annotations or somehow differently

### How to better document configuration parameters

- Current documentation hold no relative information, so we need to have a separate file called Configuration.md
- Refactor main md file to navigate to configuration

### How to better document use case and security risks

- Current document hold no relative information on use case
- Current document hold no relative infromation on security aspect
- Make a separate file Security
- Make a separate file Configuration
- Make a separate file Guide
