## Proposal for Benchmarking Cross-Service Dependencies in Microservices

Since MicroDepGraph primarily analyzes Docker configurations, it might miss finer-grained relationships that can be extracted via deeper static code analysis or runtime logs.

[Ding et al.](https://arxiv.org/abs/2310.11248)." outline a strategy to create a benchmark dataset that captures cross-code dependencies in a application-level context. They propose a method to eval LLM's cross-file analysis ability using code completion tasks in the snippets that been pinpointed cross-file context is necessary.

We can adapt their insight to create a benchmark for microservices, focusing on cross-service dependencies.

---

### 1. Define the Benchmark Objectives

- **Goal:**  
  Create a dataset that captures the dependency relationships between microservices, not only static analysis, maybe based on operational configurations (like Docker Compose) and by analyzing the source code, but also runtime behavior (logs, API calls).

- **Scope:**  
  - **Static Analysis:** Extracting API calls (Using OpenAPI/Swagger specifications to capture service interfaces), internal module imports, or references between services.
  - **Runtime Logs:** Capturing network interactions, REST/gRPC calls, or messaging events that indicate service dependencies.

- **Evaluation Metrics:**  
  - Code/Configuration completion in snippets that represent service dependencies.
  - Accuracy in identifying service dependencies.
  - Completeness of the dependency graph.
  - Ability of an LLM to answer queries about the architecture correctly.

---

### 2. Data Collection Strategies

#### A. Static Analysis

- **Tools and Techniques:**  
  - **Dependency Graph Generation:**
    Use tools like *MicroDepGraph* to create a dependency graph from the static analysis results. This tool can help visualize and export the relationships between services.
    Tools like MicroDepGraph are designed to analyze microservice dependencies from Docker configurations. Run such tools to obtain an initial dependency graph. Then we can use the output (GraphML, Neo4j database, or SVG) to devise tasks like `architecture summarization`, `dependency verification` and `question answering`.
  - **Linters/Analyzers:**  
    Use tools like *tree-sitter* or specific language parsers to extract function calls and module imports, like *Pylint* for Python, *javac* for Java, *ESLint* for JavaScript and *csc* for C# to identify cross-file dependencies.
  - **Parsing API Contracts (OpenAPI/Swagger Definitions):**  
    **Objective:**  
    Extract OpenAPI or Swagger definitions from service repositories. These files provide a structured description of the RESTful endpoints that one service might expose and that other services might consume. These contracts often exist as JSON or YAML files in a repository.
    
    **Steps:**
    
    1. **Identify API Specification Files:**  
       - Look for files named like `openapi.yaml`, `swagger.json`, or similar in each service repository.
    
    2. **Use a YAML/JSON Parser in Python:**  
       - For YAML, we can use libraries such as `PyYAML` or `ruamel.yaml`.  
       - For JSON, Python’s built-in `json` module is sufficient.
    
    3. **Extract Endpoints and Methods:**  
       - Load the API specification file, and then traverse its structure. In OpenAPI 3.0, for example, the endpoints are under the `"paths"` key.
       - For each path, extract the available HTTP methods (GET, POST, etc.) and details like parameters and response schemas.
    
    **Example Code Snippet (Python using PyYAML):**
    
    ```python
    import yaml
    
    def parse_openapi_spec(file_path):
        with open(file_path, 'r') as f:
            spec = yaml.safe_load(f)
        endpoints = {}
        for path, methods in spec.get('paths', {}).items():
            endpoints[path] = list(methods.keys())
        return endpoints
    
    # Example usage:
    spec_file = 'path/to/openapi.yaml'
    api_endpoints = parse_openapi_spec(spec_file)
    print(api_endpoints)
    ```
    
    *This script loads an OpenAPI spec and prints a dictionary mapping each endpoint to a list of HTTP methods it supports.*

  - **Extracting Code References via Custom Scripts:** 
    **Objective:**  
    Write scripts (using Python, for example) to traverse the codebase and identify lines where services are referenced, such as calls to a common HTTP client or SDK that is used across services.
    
    **Steps:**
    
    1. **Choose a Programming Language & Tooling:**  
       - For Python, use the built-in `ast` module for abstract syntax tree parsing, which helps analyze import statements and function calls.
       - For other languages, similar static analysis tools or libraries exist (like *javac* for Java, *ESLint* for JavaScript, etc.).
    
    2. **Define Patterns to Look For:**  
       - Identify common HTTP client libraries (e.g., Python’s `requests` library).
       - Write code to search for calls like `requests.get()`, `requests.post()`, etc.
    
    3. **Traverse the Codebase:**  
       - Write a script that recursively scans the repository for source code files.
       - Parse each file and extract lines or nodes (in the AST) that match the patterns.
    
    **Example Code Snippet (Python using `ast`):**
    
    ```python
    import ast
    import os
    
    class HttpCallVisitor(ast.NodeVisitor):
        def __init__(self):
            self.calls = []
    
        def visit_Call(self, node):
            # Check if the function call is a requests call
            if isinstance(node.func, ast.Attribute) and isinstance(node.func.value, ast.Name):
                if node.func.value.id == 'requests' and node.func.attr in ['get', 'post', 'put', 'delete']:
                    self.calls.append((node.lineno, node.func.attr))
            self.generic_visit(node)
    
    def extract_http_calls(file_path):
        with open(file_path, 'r') as f:
            source = f.read()
        tree = ast.parse(source, filename=file_path)
        visitor = HttpCallVisitor()
        visitor.visit(tree)
        return visitor.calls
    
    def scan_directory_for_calls(directory):
        result = {}
        for root, _, files in os.walk(directory):
            for file in files:
                if file.endswith('.py'):
                    file_path = os.path.join(root, file)
                    calls = extract_http_calls(file_path)
                    if calls:
                        result[file_path] = calls
        return result
    
    # Example usage:
    code_directory = 'path/to/microservice'
    http_calls = scan_directory_for_calls(code_directory)
    for file, calls in http_calls.items():
        print(f"{file}: {calls}")
    ```
    
    *This script scans all Python files in a directory, parses them using the AST module, and extracts lines where the `requests` library is used to make HTTP calls.*

#### B. Runtime Log Analysis (Leave it for later)

#### C. Combining Data from Multiple Sources

- **Unified Dependency Graph:**  
  - Merge static analysis results (which provide design-time dependencies) with runtime log data (which capture operational behavior). 
  - Use a common data model (e.g., a graph database like Neo4j or a GraphML format) to represent this combined dependency information.
- **Ground Truth Annotation:**
  - Manually validate a subset of the identified dependencies to create a ground truth dataset.

---

### 3. Constructing the Benchmark Dataset

#### A. Dataset Format

- **Graph Representation:**  
  We can represent the dependency graph in various formats:
  - **GraphML:** An XML-based format for graph data.
  - **Neo4j Database:** Using a labeled property graph model.
  - **JSON:** A custom JSON format that captures nodes (services) and edges (dependencies).
- **Annotation Files:**  
  Include metadata such as:
  - Service names and versions.
  - Type of dependency (e.g., REST call, messaging, direct code import).
  - Identifiers for the source code locations (e.g., file names, line numbers).
  - Confidence scores (if applicable) for automatically extracted dependencies.
  
  Based on the metadata, we can:
  - Annotate the relationships (e.g., Service A calls Service B’s `/order` endpoint) in json format.
  - Place [CURSOR_POSITION] markers in the code/configuration snippets that need cross-service knowledge to indicate where the model should fill in the gaps.
  
#### B. Benchmark Tasks

- **Code/Configuration Completion:**  
  Provide snippets of code/configuration files with missing parts (e.g., a service definition without its dependencies) and ask models to complete them. For deeper implementation of configuration completion, refer [Appendix](#appendix-adapting-crosscodeeval-for-microservices-in-configuration-files).

  For example, in the `e-commerce-microservices-sample`, the original code snippet in `users-cna-microservice/app.py`:
  ```python
  if __name__ == '__main__':
    uvicorn.run("app:app", port=9090, host='127.0.0.1', reload=True)
  ```
  can be masked to:
  ```python
  if __name__ == '__main__':
    uvicorn.run[CURSOR_POSITION]
  ```
  The model should be able to fill in the missing part with the correct port and host information, which is defined in `infra/k8s/apps/base/users/service.yaml`:
  ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: users-service
    spec:
      ports:
      - name: http
        port: 9090
      selector:
        app: users-deployment
  ```
  Similarly, we can mask the `port` and `host` in the `users-cna-microservice/Dockerfile` of the `users-cna-microservice`:
  ```dockerfile
    # Install python in the container
    FROM python:3.10.5-alpine3.16
    RUN pip install pipenv
    WORKDIR /usr/src/app
    # Copy the Pipfile
    COPY Pipfile ./
    # install the packages from the requirements.txt file in the container
    RUN pipenv install
    # expose the port that uvicorn will run the app
    [CURSOR_POSITION]
    # copy the local app/ folder to the /app fodler in the container
    COPY . .
    # execute the command python main.py (in the WORKDIR) to start the app
    CMD ["pipenv", "run", "python", "app.py"]
  ```

- **Question Answering:**  
  Pose questions such as “Which services depend on the payment service?” and provide ground truth answers.
- **Architecture Summarization:**  
  Ask models to generate a xml or graphviz representation of the architecture based on the dependency graph.
- **Dependency Verification:**  
    - In the service-level context, ask models to verify if a specific dependency exists between two services or ask for the complete dependency path between two services.
    - In the code-level context, based on the code snippet, ask models to verify if a specific function call exists in the codebase and which service it belongs to.

Define evaluation metrics (e.g., precision, recall and F1 score) for correctly identifying cross-service dependencies.

---

### Summary

To construct an evaluation benchmark that captures cross-service dependencies:

1. **Define clear objectives** (static vs. dynamic dependencies, evaluation tasks).
2. **Collect data** using both static analysis (code, API contracts) and runtime log analysis.
3. **Construct a dataset** that includes a unified dependency graph, metadata, and ground truth annotations.
4. **Design evaluation tasks** (question answering, summarization, dependency verification).

---

### Appendix: Adapting CROSSCODEEVAL for Microservices in Configuration Files

#### 1. Identify Key Dependency Elements in Config Files

- **What to Hide:**  
  In microservices config files (e.g., Docker Compose, Kubernetes YAML), key dependency information might include:
  - Service names and references (e.g., links, depends_on, network settings)
  - Environment variables or configuration values that determine inter-service communication
  - API endpoints or connection strings

---

#### 2. Create a Ground Truth

- **Extract the Complete Config:**  
  For each microservice, extract the full configuration file and mark the parts that represent inter-service dependencies.
  
- **Manual Verification:**  
  Ensure that the masked parts are indeed critical for establishing service dependencies. This forms our ground truth for evaluation.

---

#### 3. Benchmark Design Inspired by CrossCodeEval

- **Masking Strategy:**  
  Similar to CROSSCODEEVAL, where they hide a line in the source code that requires cross-file context, we can:
  - Remove or replace key lines in our config files (e.g., replace a service reference with a placeholder).
  - Ensure that the masked part is essential to understanding the dependency (for example, if Service A depends on Service B’s endpoint, remove that endpoint from Service A’s config).

- **Prompt Construction:**  
  Provide the LLM with the incomplete config file as a prompt.

- **Evaluation Metrics:**  
  Once the LLM generates a completion, compare it with the original config. Metrics can include:
  - Exact match (line-level or token-level)
  - Edit similarity (how many changes are needed to match the ground truth)
  - Validity checks using configuration validators (e.g., kubeval for Kubernetes YAML or Docker Compose linting tools)
