# Using reasoning for data validation

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

In this guide, we’ll explore how to use the o1 model, specifically o1-preview, to perform data validation through reasoning. We’ll walk through a practical example involving a synthetic medical dataset and demonstrate how to assess the model’s accuracy in identifying issues within the data.

## Overview

Data validation is a critical step in ensuring the quality and reliability of datasets, especially in sensitive fields like healthcare. Traditional validation methods often rely on predefined rules and patterns. However, advanced  models like o1 can understand context and reason about data, offering a more flexible and intelligent approach to validation.

In this tutorial, we will:
- Generate a synthetic dataset of medical data that contains inconsistencies.
- Define a function that takes in a row of data and validates its accuracy
- Run the validation process and compute accuracy metrics.
- Analyze and interpret the results.

```python
from openai import OpenAI
import json
from IPython.display import display, HTML
from sklearn.metrics import precision_score, recall_score, f1_score
from concurrent.futures import ThreadPoolExecutor, as_completed
import csv
import pandas as pd

client = OpenAI()
MODEL = 'o1-preview'
```

## Synthetic Data Generation

We will use a lot of the principles described in the [Synthetic Data Generation](https://developers.openai.com/cookbook/examples/sdg1) cookbook to create the foundation of our dataset.

We will prompt the model to generate sets of medical data for our use case. We have provided detailed instructions to the model on how to create the dataset, what format to follow, and how to fill it with inaccuracies. We also provide a few rows of sample data to get the model started. 

Each row in the dataset will have the following fields:
- Patient ID: A randomly generated patient id
- Date of Birth: Date of birth of the patient
- Gender: M/F
- Medical History: Past diagnoses
- Current Medications: Medication the patient is taking
- Allergies: Identified allergies
- Lab Results (Glucose mg/dL)
- Diagnoses: Current diagnosis
- Treatment Plan: Current treatment plan
- Is Valid: Whether or not the current row of data is valid (True/False)
- Issue: If the row of data is not valid, what the issue is

Some examples of inaccuracies that may be present in the data are:
- Prescribing medications that the patient is allergic to
- Current medications do not match medical history
- Treatment plan does not match diagnosis

````python
def generate_data():
    messages = [
        {
            "role": "user",
            "content": """
You are a helpful assistant designed to generate data. You will be given a format for the data to generate and some examples of the data.

When generating Patient IDs, use the format 'P' followed by a three-digit number (e.g., P006, P941, P319).

Intentionally make some mistakes in the data generation and document them in the appropriate columns ('Is Valid' and 'Issue') if the row of data is invalid.

The types of mistakes to include are:

- **Allergy Contradictions**: Prescribing a medication that the patient is allergic to (e.g., prescribing Penicillin to a patient allergic to Penicillin).
- **Medical History and Medication Mismatch**: A patient with a medical condition not receiving appropriate medication (e.g., a diabetic patient not prescribed any diabetes medication).
- **Lab Results and Diagnosis Mismatch**: Lab results that do not support the diagnosis (e.g., normal glucose levels but diagnosed with Diabetes Type 2).
- **Other Plausible Mistakes**: Any other realistic errors that could occur in medical records, such as incorrect gender entries, impossible dates of birth, or inconsistent treatment plans.

Ensure that when 'Is Valid' is 'False', the 'Issue' column clearly explains the problem.

Return 100 rows of data for the user. Your response should strictly be in the format of a valid CSV.

Generate Synthetic Medical Records Dataset with the following columns:
    - Patient ID: A randomly generated patient id
    - Date of Birth: Date of birth of the patient
    - Gender: M/F
    - Medical History: Past diagnoses
    - Current Medications: Medication the patient is taking
    - Allergies: Identified allergies
    - Lab Results (Glucose mg/dL)
    - Diagnoses: Current diagnosis
    - Treatment Plan: Current treatment plan
    - Is Valid: Whether or not the current row of data is valid (True/False)
    - Issue: If the row of data is not valid, what the issue is

Patient ID,Date of Birth,Gender,Medical History,Current Medications,Allergies,Lab Results (Glucose mg/dL),Diagnoses,Treatment Plan,Is Valid,Issue
P001,1980-05-14,M,Hypertension,Lisinopril,None,110,Hypertension,Continue Lisinopril,True,
P002,1975-11-30,F,Diabetes Type 2,Metformin,Penicillin,90,Diabetes Type 2,Continue Metformin,True,
P003,1990-07-22,F,Asthma,Albuterol,Aspirin,85,Asthma,Prescribe Albuterol,True,
P004,2000-03-10,M,None,Amoxicillin,Penicillin,95,Infection,Prescribe Amoxicillin,False,Prescribed Amoxicillin despite Penicillin allergy
P005,1985-09-18,F,Hyperlipidemia,Atorvastatin,None,200,Hyperlipidemia,Continue Atorvastatin,True,
P006,1978-12-05,M,Hypertension; Diabetes Type 2,Lisinopril; Insulin,None,55,Diabetes Type 2,Adjust insulin dosage,False,Low glucose level not properly addressed
            """
        }
    ]

    response = client.chat.completions.create(
        model=MODEL,
        messages=messages
    )

    return response.choices[0].message.content.replace('```csv', '').replace('```', '')
````

```python
# Generate data three times using the existing dataGeneration function
generated_data = []
data = generate_data()
generated_data.extend(data.strip().split('\n'))

# Append the generated data to the medicalData.csv file
with open('../data/medicalData.csv', 'a', newline='') as csvfile:
    csvwriter = csv.writer(csvfile)
    for row in generated_data:
        csvwriter.writerow(row.split(','))

print("Synthetic data generation and appending completed.")
```

```text
Synthetic data generation and appending completed.
```

## Data Validation

Now that we have our dataset prepared, we will prompt the reasoning model to review each row of data and determine whether or not it contains an issue. We will ask the model to output whether or not there is an issue in the data and then offer an explanation of the issue.

Once we have the model determine its list of invalid data, we will pass those results on to a model grader to assess two metrics:
- Accuracy of the model's ability correctly identify issues with the data
- For the subset of data that issues have been correctly identified, what is the accuracy of the model in identifying the issue at hand

Given that this task is much more narrow, we can use the faster gpt-4o model to calculate the accuracy.

REMINDER: Given that these models are still in beta, rate limits will be significantly reduced. Please adjust the number of concurrent workers accordingly.

````python
def validate_data(input_data):
    messages = [
        {
            "role": "user",
            "content": f"""
You are a helpful assistant designed to validate the quality of medical datasets. You will be given a single row of medical data, and your task is to determine whether the data is valid.

- Carefully analyze the data for any inconsistencies, contradictions, missing values, or implausible information.
- Consider the logical relationships between different fields (e.g., treatments should be appropriate for the diagnoses, medications should not conflict with allergies, lab results should be consistent with diagnoses, etc.).
- Use your general medical knowledge to assess the validity of the data.
- Focus solely on the information provided without making assumptions beyond the given data.

**Return only a JSON object** with the following two properties:

- `"is_valid"`: a boolean (`true` or `false`) indicating whether the data is valid.
- `"issue"`: if `"is_valid"` is `false`, provide a brief explanation of the issue; if `"is_valid"` is `true`, set `"issue"` to `null`.

Both JSON properties must always be present.

Do not include any additional text or explanations outside the JSON object.

MEDICAL DATA:
{input_data}
            """
        }
    ]

    response = client.chat.completions.create(
        model=MODEL,
        messages=messages
    )

    response_content = response.choices[0].message.content.replace('```json', '').replace('```', '').strip()
    
    try:
        if isinstance(response_content, dict):
            response_dict = response_content
        else:
            response_dict = json.loads(response_content)
        return response_dict
    except json.JSONDecodeError as e:
        print(f"Failed to decode JSON response: {response_content}")
        raise e
````

```python
# Read the CSV file and exclude the last two columns
input_data = []
with open('../data/medicalData.csv', 'r') as file:
    reader = csv.reader(file)
    headers = next(reader)
    for row in reader:
        input_data.append(row[:-2])  # Exclude "Is Valid" and "Issue" columns

# Initialize lists to store true labels
true_is_valid = []
true_issues = []

# Extract true labels from the CSV file
with open('../data/medicalData.csv', 'r') as file:
    reader = csv.reader(file)
    headers = next(reader)
    for row in reader:
        true_is_valid.append(row[-2] == 'True')
        true_issues.append(row[-1])

# Function to validate a single row of data
def validate_row(row):
    input_str = ','.join(row)
    result_json = validate_data(input_str)
    return result_json

# Validate data rows and collect results
pred_is_valid = [False] * len(input_data)
pred_issues = [''] * len(input_data)

with ThreadPoolExecutor() as executor:
    futures = {executor.submit(validate_row, row): i for i, row in enumerate(input_data)}
    
    for future in as_completed(futures):
        i = futures[future]  # Get the index of the current row
        result_json = future.result()
        pred_is_valid[i] = result_json['is_valid']
        pred_issues[i] = result_json['issue']
```

Now that we have the model's results, we can compare it against the source of truth and determine the system's accuracy

```python
# Convert predicted and true 'is_valid' labels to boolean if they aren't already
pred_is_valid_bool = [bool(val) if isinstance(val, bool) else val == 'True' for val in pred_is_valid]
true_is_valid_bool = [bool(val) if isinstance(val, bool) else val == 'True' for val in true_is_valid]

# Calculate precision, recall, and f1 score for the 'is_valid' prediction
precision = precision_score(true_is_valid_bool, pred_is_valid_bool, pos_label=True)
recall = recall_score(true_is_valid_bool, pred_is_valid_bool, pos_label=True)
f1 = f1_score(true_is_valid_bool, pred_is_valid_bool, pos_label=True)

# Initialize issue_matches_full with False
issue_matches_full = [False] * len(true_is_valid)
```

```python
print(f"Precision: {precision:.2f}")
print(f"Recall: {recall:.2f}")
print(f"F1: {f1:.2f}")
```

```text
Precision: 0.82
Recall: 0.87
F1: 0.84
```

## Issue Identification

We will now determine the model's ability to accurately classify the issue in the data

```python
def validate_issue(model_generated_answer, correct_answer):
    messages = [
        {
            "role": "user",
            "content": f"""
You are a medical expert assistant designed to validate the quality of an LLM-generated answer.

The model was asked to review a medical dataset row to determine if the data is valid. If the data is not valid, it should provide a justification explaining why.

Your task:

    •	Compare the model-generated justification with the correct reason provided.
    •	Determine if they address the same underlying medical issue or concern, even if phrased differently.
    •	Focus on the intent, medical concepts, and implications rather than exact wording.

Instructions:

    •	If the justifications have the same intent or address the same medical issue, return True.
    •	If they address different issues or concerns, return False.
    •	Only respond with a single word: True or False.

Examples:

    1.	Example 1:
    •	Model Generated Response: “The patient is allergic to penicillin”
    •	Correct Response: “The patient was prescribed penicillin despite being allergic”
    •	Answer: True
    2.	Example 2:
    •	Model Generated Response: “The date of birth of the patient is incorrect”
    •	Correct Response: “The patient was prescribed penicillin despite being allergic”
    •	Answer: False


Model Generated Response: {model_generated_answer}
Correct Response:  {correct_answer}
            """
        }
    ]

    response = client.chat.completions.create(
        model="o1-preview",
        messages=messages
    )

    result = response.choices[0].message.content

    return result
```

```python
# Validate issues for rows where both true and predicted 'is_valid' are False
validation_results = []

with ThreadPoolExecutor() as executor:
    futures = {
        executor.submit(validate_issue, pred_issues[i], true_issues[i]): i
        for i in range(len(pred_is_valid_bool))
        if not pred_is_valid_bool[i] and not true_is_valid_bool[i]
    }
    
    for future in as_completed(futures):
        i = futures[future]  # Get the original index
        issue_match = future.result()
        issue_matches_full[i] = (issue_match == 'True')
        validation_results.append({
            "index": i,
            "predicted_issue": pred_issues[i],
            "true_issue": true_issues[i],
            "issue_match": issue_matches_full[i]
        })
    
    # Calculate issue accuracy
    issue_accuracy = sum([i['issue_match'] for i in validation_results]) / len(validation_results)
    
    # Store the results in the dictionary
    model_results = {
        "precision": precision,
        "recall": recall,
        "f1": f1,
        "issue_accuracy": issue_accuracy
    }

# Create a DataFrame to store the results
df_results = pd.DataFrame([model_results])

# Create a DataFrame to store the validation results for each row
df_validation_results = pd.DataFrame(validation_results)
```

Below we'll display the subset of rows that we correctly identified contained an issue. For each row, we'll show the predicted vs. true issue and whether or not there is a match

```python
def display_formatted_dataframe(df):
    def format_text(text):
        return text.replace('\n', '<br>')

    df_formatted = df.copy()
    df_formatted['predicted_issue'] = df_formatted['predicted_issue'].apply(format_text)
    df_formatted['true_issue'] = df_formatted['true_issue'].apply(format_text)
    
    display(HTML(df_formatted.to_html(escape=False, justify='left')))
    
display_formatted_dataframe(pd.DataFrame(validation_results))
```

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: left;">
      <th></th>
      <th>index</th>
      <th>predicted_issue</th>
      <th>true_issue</th>
      <th>issue_match</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>39</td>
      <td>Amoxicillin is prescribed to a patient with Penicillin allergy.</td>
      <td>Prescribed Amoxicillin despite Penicillin allergy</td>
      <td>True</td>
    </tr>
    <tr>
      <th>1</th>
      <td>50</td>
      <td>Patient diagnosed with Type 1 Diabetes is not on any medications and the treatment field lists the diagnosis instead of appropriate treatment.</td>
      <td>Diabetes Type 1 patient not receiving insulin</td>
      <td>True</td>
    </tr>
    <tr>
      <th>2</th>
      <td>51</td>
      <td>Lab result of 300 indicates hyperglycemia but no diagnosis or treatment is recorded.</td>
      <td>Extremely high glucose level not diagnosed or treated</td>
      <td>True</td>
    </tr>
    <tr>
      <th>3</th>
      <td>26</td>
      <td>The patient is being prescribed penicillin despite having an allergy to penicillin.</td>
      <td>Prescribed Penicillin despite Penicillin allergy</td>
      <td>True</td>
    </tr>
    <tr>
      <th>4</th>
      <td>31</td>
      <td>The patient's age (88) is inconsistent with the date of birth (1996-11-05).</td>
      <td>Osteoporosis patient not receiving treatment</td>
      <td>False</td>
    </tr>
    <tr>
      <th>5</th>
      <td>24</td>
      <td>The 'Treatment Plan' field should not be 'Depression'; it should specify the treatment prescribed for depression.</td>
      <td>Depression patient not receiving treatment</td>
      <td>True</td>
    </tr>
    <tr>
      <th>6</th>
      <td>3</td>
      <td>Patient is allergic to Penicillin but is prescribed Amoxicillin.</td>
      <td>Prescribed Amoxicillin despite Penicillin allergy</td>
      <td>True</td>
    </tr>
    <tr>
      <th>7</th>
      <td>28</td>
      <td>The treatment field contains 'Asthma', which is a diagnosis, not a treatment.</td>
      <td>Asthma patient not prescribed any medication</td>
      <td>False</td>
    </tr>
    <tr>
      <th>8</th>
      <td>7</td>
      <td>Patient with asthma and low lab result (100) is treated only with lifestyle modifications without medications, which is inappropriate.</td>
      <td>Asthma patient not prescribed any medication</td>
      <td>True</td>
    </tr>
    <tr>
      <th>9</th>
      <td>16</td>
      <td>The patient's age (86) does not match the date of birth (1955-10-10).</td>
      <td>COPD patient not receiving treatment</td>
      <td>False</td>
    </tr>
    <tr>
      <th>10</th>
      <td>53</td>
      <td>The age provided (92) is inconsistent with the date of birth (1983-08-19).</td>
      <td>Depression patient not receiving treatment</td>
      <td>False</td>
    </tr>
    <tr>
      <th>11</th>
      <td>23</td>
      <td>Treatment field incorrectly lists 'Hyperlipidemia' instead of an appropriate treatment for the diagnosis.</td>
      <td>Hyperlipidemia patient not prescribed any medication</td>
      <td>True</td>
    </tr>
    <tr>
      <th>12</th>
      <td>13</td>
      <td>Patient is allergic to sulfa drugs but is prescribed Sulfamethoxazole, which is a sulfa drug.</td>
      <td>Prescribed Sulfa drug despite Sulfa allergy</td>
      <td>True</td>
    </tr>
    <tr>
      <th>13</th>
      <td>98</td>
      <td>The patient is prescribed Penicillin despite having a Penicillin allergy.</td>
      <td>Prescribed Penicillin despite Penicillin allergy</td>
      <td>True</td>
    </tr>
    <tr>
      <th>14</th>
      <td>9</td>
      <td>Patient has a medication allergy to Penicillin but is prescribed Penicillin.</td>
      <td>Prescribed Penicillin despite Penicillin allergy</td>
      <td>True</td>
    </tr>
    <tr>
      <th>15</th>
      <td>85</td>
      <td>Treatment field contains 'Hyperlipidemia', which is a diagnosis, not a treatment.</td>
      <td>Hyperlipidemia patient not prescribed any medication</td>
      <td>False</td>
    </tr>
    <tr>
      <th>16</th>
      <td>18</td>
      <td>Prescribed treatment (Aspirin) is not appropriate for the diagnosis of infection.</td>
      <td>Prescribed Aspirin despite Aspirin allergy; high glucose level not addressed</td>
      <td>False</td>
    </tr>
    <tr>
      <th>17</th>
      <td>70</td>
      <td>Treatment field contains a diagnosis 'Osteoporosis' instead of a treatment.</td>
      <td>Osteoporosis patient not receiving treatment</td>
      <td>True</td>
    </tr>
    <tr>
      <th>18</th>
      <td>57</td>
      <td>Patient is allergic to Penicillin but is being prescribed Amoxicillin, which is contraindicated.</td>
      <td>Prescribed Amoxicillin despite Penicillin allergy</td>
      <td>True</td>
    </tr>
    <tr>
      <th>19</th>
      <td>80</td>
      <td>Treatment field incorrectly lists 'Diabetes Type 2' instead of a valid treatment plan.</td>
      <td>Diabetes Type 2 patient not receiving medication</td>
      <td>True</td>
    </tr>
    <tr>
      <th>20</th>
      <td>87</td>
      <td>Treatment plan includes prescribing Amoxicillin, which the patient is allergic to.</td>
      <td>Prescribed Amoxicillin despite Penicillin allergy</td>
      <td>True</td>
    </tr>
    <tr>
      <th>21</th>
      <td>37</td>
      <td>Treatment field contains 'Hyperlipidemia', which is a diagnosis, not a treatment.</td>
      <td>Hyperlipidemia patient not prescribed any medication</td>
      <td>False</td>
    </tr>
    <tr>
      <th>22</th>
      <td>95</td>
      <td>Treatment is listed as 'Asthma', which is not an appropriate treatment for the diagnosis.</td>
      <td>Asthma patient not prescribed any medication</td>
      <td>True</td>
    </tr>
    <tr>
      <th>23</th>
      <td>96</td>
      <td>Treatment field lists 'Hyperlipidemia', which is not an appropriate treatment.</td>
      <td>Hyperlipidemia patient not prescribed any medication</td>
      <td>False</td>
    </tr>
    <tr>
      <th>24</th>
      <td>59</td>
      <td>Treatment field contains 'Anemia', which is not a valid treatment.</td>
      <td>Anemia patient not receiving treatment</td>
      <td>False</td>
    </tr>
    <tr>
      <th>25</th>
      <td>5</td>
      <td>Age does not match date of birth</td>
      <td>Low glucose level not properly addressed</td>
      <td>False</td>
    </tr>
  </tbody>
</table>

```python
# Display the DataFrame
print(df_results)
```

```text
   precision    recall       f1  issue_accuracy
0   0.818182  0.870968  0.84375        0.615385
```

## Conclusion

We can see from the results here that we're able to generate a high precision/recall for issue identification as well as decent accuracy for pinpointing the exact issue in the data.

This should help streamline data validation for eval sets across a variety of domains.