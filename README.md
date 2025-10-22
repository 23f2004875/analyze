# Data Processing Project with Python, Pandas, and GitHub Actions

This project demonstrates a simple data processing workflow using Python and Pandas, includes code quality checks with Ruff, and automates the process with GitHub Actions for CI/CD and result publishing via GitHub Pages.

## Table of Contents
- [Project Overview](#project-overview)
- [Files in this Project](#files-in-this-project)
- [Setup and Local Execution](#setup-and-local-execution)
- [Understanding `execute.py`](#understanding-executepy)
  - [The Non-Trivial Error and Its Fix](#the-non-trivial-error-and-its-fix)
  - [Final `execute.py` Code](#final-executepy-code)
- [Data Files](#data-files)
  - [`data.xlsx`](#dataxlsx)
  - [`data.csv`](#datacsv)
- [GitHub Actions Workflow](#github-actions-workflow)
  - [`ci.yml` Content](#ciyml-content)
  - [GitHub Pages Deployment](#github-pages-deployment)
- [License](#license)

## Project Overview

The core of this project is to process sales data from an Excel file (`data.xlsx`), convert it into a CSV format (`data.csv`), perform an aggregation using a Python script (`execute.py`), and output the processed data as a JSON file (`result.json`). The entire pipeline is automated using GitHub Actions, ensuring code quality with Ruff and publishing the final `result.json` to GitHub Pages.

## Files in this Project

- `index.html`: A simple, responsive web page describing the project.
- `README.md`: This document, detailing the project, setup, and code.
- `LICENSE`: The MIT License for this project.
- `execute.py`: The Python script responsible for data processing.
- `data.xlsx`: The original Excel data file.
- `data.csv`: The converted CSV data file (derived from `data.xlsx` and committed).
- `.github/workflows/ci.yml`: The GitHub Actions workflow definition.
- `result.json`: (Not committed) The output of `execute.py`, generated and published by the CI pipeline.

## Setup and Local Execution

To run this project locally, ensure you have Python 3.11+ and pip installed.

1.  **Clone the repository:**
    ```bash
    git clone https://github.com/your-username/your-repo-name.git
    cd your-repo-name
    ```

2.  **Install dependencies:**
    ```bash
    pip install pandas==2.3.0 ruff openpyxl
    ```
    (`openpyxl` is needed if you need to manually convert `data.xlsx` to `data.csv` locally, though `data.csv` is committed.)

3.  **Ensure `data.csv` exists:**
    If `data.csv` is missing, you can generate it from `data.xlsx`:
    ```bash
    python -c "import pandas as pd; pd.read_excel('data.xlsx').to_csv('data.csv', index=False)"
    ```

4.  **Run the data processing script:**
    ```bash
    python execute.py
    ```
    This will generate `result.json` in your project root.

5.  **Run Ruff linter:**
    ```bash
    ruff check .
    ```

## Understanding `execute.py`

The `execute.py` script reads `data.csv`, aggregates sales amounts and counts by product category, and outputs the summary into `result.json`.

### The Non-Trivial Error and Its Fix

Initially, `execute.py` contained a `KeyError` due to a mismatch in the column name used for grouping. The original code mistakenly used `'category'` (lowercase) while the actual column header in `data.csv` (and `data.xlsx`) is `'Category'` (uppercase). This is a common oversight in data processing that can lead to runtime errors.

**Original problematic line (conceptual):**
```python
# summary = df.groupby('category')['Amount'].sum().reset_index() # This would raise a KeyError
```

**The fix involved:**
1.  Correcting the column name to `'Category'`.
2.  Adding robust error handling for `FileNotFoundError`, `KeyError`, and general exceptions.
3.  Ensuring the 'Amount' column is properly converted to a numeric type, handling potential non-numeric entries gracefully.
4.  Enhancing the output by also counting items per category.

### Final `execute.py` Code

```python
import pandas as pd
import json

def process_data(file_path):
    """
    Reads a CSV file, processes it, and returns a summary as a list of dictionaries.
    
    Args:
        file_path (str): The path to the input CSV file.
        
    Returns:
        list: A list of dictionaries, where each dictionary represents a summary row.
    """
    try:
        df = pd.read_csv(file_path)
        
        # Ensure 'Amount' column is numeric, coercing errors to NaN and filling with 0
        df['Amount'] = pd.to_numeric(df['Amount'], errors='coerce').fillna(0)
        
        # Corrected column name: 'Category' instead of 'category' to fix KeyError
        summary = df.groupby('Category')['Amount'].sum().reset_index()
        
        # Add item count per category for more comprehensive summary
        count_summary = df.groupby('Category').size().reset_index(name='ItemCount')
        
        # Merge the two summaries
        final_summary = pd.merge(summary, count_summary, on='Category')
        
        return final_summary.to_dict(orient='records')
        
    except FileNotFoundError:
        print(f"Error: The file {file_path} was not found. Please ensure 'data.csv' exists.")
        return []
    except KeyError as e:
        print(f"Error: Missing expected column in CSV: {e}. Please check 'Category' and 'Amount' headers.")
        return []
    except Exception as e:
        print(f"An unexpected error occurred during data processing: {e}")
        return []

if __name__ == "__main__":
    output_file = 'result.json'
    result_data = process_data('data.csv')
    
    if result_data:
        with open(output_file, 'w') as f:
            json.dump(result_data, f, indent=4)
        print(f"Successfully processed data and saved results to {output_file}")
    else:
        print("No data processed or an error occurred. No result.json was generated.")
```

## Data Files

### `data.xlsx`

This is the original Excel spreadsheet containing raw sales data. It has the following columns: `OrderID`, `Product`, `Category`, `Amount`, `Date`.

### `data.csv`

The `data.xlsx` file is converted into `data.csv` for easier programmatic access and to align with the `execute.py` script's `pd.read_csv` function. This `data.csv` file is committed to the repository.

**Sample `data.csv` content:**

```csv
OrderID,Product,Category,Amount,Date
1001,Laptop,Electronics,1200.00,2023-01-05
1002,Mouse,Electronics,25.50,2023-01-05
1003,Keyboard,Electronics,75.00,2023-01-06
1004,Desk Chair,Furniture,150.00,2023-01-07
1005,Coffee Table,Furniture,90.00,2023-01-07
1006,Monitor,Electronics,300.00,2023-01-08
1007,Book,Books,20.00,2023-01-09
1008,Pen Set,Office Supplies,15.00,2023-01-09
1009,Printer,Electronics,250.00,2023-01-10
1010,Sofa,Furniture,500.00,2023-01-11
```

## GitHub Actions Workflow

A GitHub Actions workflow (`.github/workflows/ci.yml`) is set up to automate the following tasks on every push to the `main` branch:

-   **Code Linting:** Runs `ruff` to ensure code quality and style compliance.
-   **Data Processing:** Executes `execute.py` to process `data.csv` and generate `result.json`.
-   **Output Publication:** Publishes the `result.json` file to GitHub Pages.

### `ci.yml` Content

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
  workflow_dispatch: # Allows manual triggering of the workflow

jobs:
  build_and_deploy:
    runs-on: ubuntu-latest

    permissions:
      contents: read # Required to checkout the repository and read files
      pages: write # Required to deploy to GitHub Pages
      id-token: write # Required for OpenID Connect authentication to GitHub Pages

    environment:
      name: github-pages # Explicitly specify the environment for Pages deployment
      url: ${{ steps.deployment.outputs.page_url }} # URL of the deployed page

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Set up Python 3.11
      uses: actions/setup-python@v5
      with:
        python-version: '3.11' # Ensures Python 3.11+ as requested

    - name: Install dependencies (Pandas 2.3.0, Ruff)
      run: |
        python -m pip install --upgrade pip
        pip install pandas==2.3.0 ruff

    - name: Run Ruff Linter
      run: ruff check .
      continue-on-error: true # Continues workflow even if linting issues are found

    - name: Run execute.py and generate result.json
      # execute.py is designed to read data.csv and output result.json
      run: python execute.py
      
    - name: Create output directory for GitHub Pages
      run: |
        mkdir -p _site # Create a directory to hold static content for Pages
        mv result.json _site/result.json # Move the generated result.json into this directory

    - name: Upload Pages artifact
      uses: actions/upload-pages-artifact@v3
      with:
        path: '_site' # Upload the _site directory as an artifact for Pages

    - name: Deploy to GitHub Pages
      id: deployment
      uses: actions/deploy-pages@v4
```

### GitHub Pages Deployment

Upon successful completion of the `build_and_deploy` job, the `result.json` file will be published to your repository's GitHub Pages. You can access it at a URL similar to: `https://<your-username>.github.io/<your-repository-name>/result.json`.

## License

This project is licensed under the MIT License. See the `LICENSE` file for full details.