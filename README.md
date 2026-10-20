# Lab 2: Building and Packaging an Application with Gradle

## Objective
Build, test, run, and verify the Calculator application using Gradle.

## Step 1: Install Java, Gradle, and Git

Install the required tools:

```bash
sudo apt update
sudo apt install -y openjdk-21-jdk gradle git
```

Verify the installations:

```bash
java -version
gradle -version
git --version
```

## Step 2: Clone the Source Code

Clone the project repository and navigate to the project directory:

```bash
cd ~/ivolve
git clone https://github.com/Ibrahim-Adel15/CalculatorGradleApp.git
cd CalculatorGradleApp
```

List the project files:

```bash
ls -la
```

The project contains `build.gradle`, `settings.gradle`, and the `src` directory.

## Step 3: Run Unit Tests

Run the unit tests:

```bash
gradle test
```

To clean previous build outputs and run the tests again:

```bash
gradle clean test
```

**Result:** `BUILD SUCCESSFUL`

## Step 4: Build the Application

Build the application and generate the JAR artifact:

```bash
gradle build
```

**Result:** `BUILD SUCCESSFUL`

## Step 5: Verify the Generated Artifact

List the generated JAR files:

```bash
ls -lh build/libs/
```

Expected artifact:

```text
build/libs/calculator.jar
```

## Step 6: Run the Application

Execute the generated JAR file:

```bash
java -jar build/libs/calculator.jar
```

Example input:

```text
Enter first number: 1234
Enter second number: 1
```

## Step 7: Verify the Application

The application returned the following results:

```text
a = 1234.0, b = 1.0
Sum: 1235.0
Subtract: 1233.0
Multiply: 1234.0
Divide: 1234.0
```

The results were checked and found to be mathematically correct.


