# Lab 1: Building and Packaging a Java Application with Maven
## Prerequisites
- Java JDK 17
- Apache Maven
- Git

## Steps

### 1. Clone the Repository
```bash
git clone https://github.com/lbrahim-Adel15/CalculatorMavenApp.git
cd CalculatorMavenApp
```

### 2. Verify Java and Maven
```bash
java -version
mvn -version
```

### 3. Run Unit Tests
```bash
mvn test
```
**Result:** All 5 tests passed with 0 failures and 0 errors.

### 4. Build and Package the Application
```bash
mvn clean package
```
This command compiles the source code, runs the tests, and packages the application into a JAR file.

**Generated artifact:**
```text
target/calculator.jar
```

### 5. Run the Application
```bash
java -jar target/calculator.jar
```

### 6. Verify the Build
```bash
ls -lh target/calculator.jar
cat target/surefire-reports/*.txt
```

## Results
- Java and Maven installed successfully.
- Unit tests executed successfully: 5 tests passed.
- Application packaged as `target/calculator.jar`.
- Build completed successfully with `BUILD SUCCESS`.
