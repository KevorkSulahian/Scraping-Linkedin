##  Exploration of Job Application Automation with Data Scraping
### Introduction
In today's competitive job market, efficiency is key. Automating repetitive tasks is one way to improve this efficiency. While full automation might not be possible due to varying application requirements and human interactions, partial automation can certainly help ease the burden. This article explores the technical aspects of automating job applications, focusing on the theoretical concept rather than practical application to ensure we abide by websites' terms of service.

### What is Web Scraping?
Web scraping is a technique for extracting information from websites. It automates the manual process of collecting data from the web, using various programming languages and tools.

The first step in any web scraping project is to import the necessary Python packages and set up a web driver. The web driver allows us to programmatically control a web browser, enabling interactions with web pages just as a human user would.

For example, we'll be using Selenium, a powerful tool for controlling a web browser through programs and automating browser automation. We'll also use pandas for data manipulation and time for pause delays.

![Image](image.webp)



```python
#Import Packages
from selenium import webdriver # Web scraping library to control browser interaction
import pandas as pd 
from time import sleep
import requests
from lxml import html
import traceback

# The following imports are included but not used in this code snippet. 
# They can be useful for advanced functionalities like waiting for certain conditions.
# from selenium.webdriver.support.select import Select 
# from selenium.webdriver.support.ui import WebDriverWait 
# from selenium.webdriver.common.by import By 
# from selenium.webdriver.support import expected_conditions as EC 

# The import below is not used but can be important if you want to add Chrome options.
# from selenium.webdriver.chrome.options import Options 

# Path to webdriver, can be chrome or anything else you have
# Note: Replace the empty string with the path to your webdriver executable.
driver = webdriver.Chrome(executable_path="")

```

driver = webdriver.Chrome(executable_path=""): Initializes a new Chrome browser window controlled by the webdriver. Replace the empty string with the path to your ChromeDriver executable. Make sure you've downloaded the appropriate ChromeDriver version compatible with your Chrome browser.

### Navigating to LinkedIn and Logging In
Once the necessary packages are imported and the webdriver is set up, the next step is to navigate to LinkedIn's login page. We will enter the username and password programmatically and then click the "Sign In" button.



```python
# Define the URL for LinkedIn login
url1='https://www.linkedin.com/login'

# Implicitly wait for 1 second; gives the browser time to load scripts
driver.implicitly_wait(1)

# Navigate to the LinkedIn login page
driver.get(url1)

# Locate email field by its id and input email
email_field = driver.find_element_by_id('username')
email_field.send_keys('') # Replace empty quotes with your email
print('- Finish keying in email')
sleep(1) # Pause for 1 second

# Locate password field by its name attribute and input password
password_field = driver.find_element_by_name('session_password')
password_field.send_keys('') # Replace empty quotes with your password
print('- Finish keying in password')
sleep(1) # Pause for 1 second

# Locate the sign-in button by its XPath and click it
signin_field = driver.find_element_by_xpath('//*[@id="organic-div"]/form/div[3]/button')
signin_field.click()
sleep(1) # Pause for 1 second

```

### Fetching Company Names You Would Like to Apply To
After successfully logging in, you might want to fetch a list of companies where you'd like to apply. For this example, we'll be scraping a list of hedge fund managers from the SWFI Institute website and saving the names to a CSV file.


```python
# URL of the page with a list of companies you're interested in
url = "https://www.swfinstitute.org/fund-manager-rankings/hedge-fund-manager"

# Fetch the webpage
response = requests.get(url)

# Parse the HTML content
tree = html.fromstring(response.content)

# Extract company names using XPath
companies = tree.xpath('//*[contains(concat( " ", @class, " " ), concat( " ", "table-striped", " " ))]//a/text()')

# Save the list of companies to a CSV file
pd.DataFrame(companies, columns=["Company"]).to_csv("companies.csv", index=False)

# Read the list back into a DataFrame to confirm
df = pd.read_csv("companies.csv")
df.head()
```

### Cleaning Company Names and Finding LinkedIn Pages
Once we have a list of companies from our CSV file, our next objective is to search for these companies on LinkedIn. However, company names can often include designations like 'LLC', 'Inc.', etc., which might interfere with our search. So, the first step is to clean up these names.

After that, we programmatically visit LinkedIn's company search page for each cleaned company name, scrape the LinkedIn URL of the company, and store these URLs for later use.


```python
def clean_company_name(company_name):
    # List of extra stuff we want to remove from the company name
    extra_stuff = ['LLC', 'LP', 'Inc.', 'Co.', 'Corp.', 'Ltd.', 'LLP', 'PLC', 'AG', 'AB', 'BV', 'GmbH']
    
    # Remove extra stuff from the name
    cleaned_name = ' '.join(word for word in company_name.split() if word not in extra_stuff)
    
    # Remove any commas
    cleaned_name = cleaned_name.replace(',', '')
    
    # Replace spaces with URL-encoded spaces (%20) for the URL
    cleaned_name = cleaned_name.replace(' ', '%20')
    
    return cleaned_name

# List to hold company LinkedIn URLs
company_links = []

# Iterate through DataFrame rows
for i, row in df.iterrows():
    # Clean the company name
    clean_name = clean_company_name(row['Company'])
    
    # Generate LinkedIn search URL for the cleaned company name
    company_search_page = f"https://www.linkedin.com/search/results/companies/?keywords={clean_name}"
    print(company_search_page)
    
    # Navigate to LinkedIn search page
    driver.get(company_search_page)
    sleep(2) # Wait 2 seconds for the page to load
    
    # Try to find the LinkedIn URL of the company
    try:
        # XPath for most common scenario
        element = driver.find_element_by_xpath('/html/body/div[5]/div[3]/div[2]/div/div[1]/main/div/div/div[2]/div/ul/li[1]/div/div/div[2]/div[1]/div[1]/div/span/span/a')
    except:
        try:
            # Backup XPath for less common scenario
            element = driver.find_element_by_xpath('/html/body/div[5]/div[3]/div[2]/div/div[1]/main/div/div/div[3]/div/ul/li/div/div/div[2]/div[1]/div[1]/div/span/span/a')
        except:
            # If both fail, skip to next iteration
            continue
            
    # Extract the LinkedIn URL
    link = element.get_attribute("href")
    
    # Append the URL to our list
    company_links.append(link)

```

After collecting the LinkedIn URLs of the companies you are interested in, it's a good idea to save this data locally. This way, you can reuse the list without having to scrape LinkedIn again, saving both time and computational resources.

In Python, one of the ways to save and load objects is by using the pickle module, which allows you to serialize (save) and deserialize (load) Python objects to and from files.


```python
import pickle

# Saving the list to a file
filename = "company_list.pkl"
# Uncomment the following lines to actually save the list
# with open(filename, "wb") as file:
#     pickle.dump(company_links, file)

# Loading the list from the file
with open(filename, "rb") as file:
    loaded_list = pickle.load(file)

# Printing the loaded list to confirm
print(loaded_list)

```

### Collecting Job Posting URLs from Company Pages
After gathering LinkedIn URLs for companies you are interested in, the next step is to explore their job listings. We will navigate to each company's jobs page, scrape the URLs of individual job listings, and store them for later use.

Here is how to collect job posting URLs from LinkedIn company pages:


```python
# List to hold job LinkedIn URLs
job_links = []

# Iterate through each company's LinkedIn URL in the loaded list
for company_url in loaded_list:
    # Navigate to the 'jobs' section of the company's LinkedIn page
    driver.get(company_url + "jobs")
    
    try:
        # Try to find the link to all job listings for the company
        element = driver.find_element_by_xpath('/html/body/div[5]/div[3]/div/div[2]/div/div[2]/main/div[2]/div/section/div/div/a')
    except:
        try:
            # If the above fails, try another XPath pattern
            element = driver.find_element_by_xpath('/html/body/div[4]/div[3]/div/div[2]/div/div[2]/main/div[2]/div/section/div/div/a')
        except:
            # If both XPaths fail, skip to the next company
            continue

    # Extract the URL of the page containing all job listings
    link = element.get_attribute("href")

    # Navigate to the extracted link
    driver.get(link)
    sleep(1)  # Wait for a second to ensure the page loads
    
    # Find the ul element containing the job listings
    ul_element = driver.find_element_by_xpath("/html/body/div[5]/div[3]/div[4]/div/div/main/div/div[1]/div/ul")
    
    # Extract all the li elements (each represents a job listing) within the ul
    li_elements = ul_element.find_elements_by_tag_name("li")

    # Iterate over each li element to extract job links
    for element in li_elements:
        link_element = element.find_elements_by_class_name("job-card-list__title")
        if link_element:
            job_links.append(link_element[0].get_attribute("href"))

# Filename to store the list of job URLs
filename = "job_links.pkl"

# Saving the list to a file (uncomment when ready to save)
# with open(filename, "wb") as file:
#     pickle.dump(job_links, file)

# Loading the list back from the saved file
with open(filename, "rb") as file:
    loaded_job_links = pickle.load(file)

# Printing the loaded list to ensure it loaded correctly
print(loaded_job_links)
```

### Extracting Job Details
We have previously managed to compile lists of both company and job URLs. The next vital step is to extract relevant details from the job postings themselves. This will include information like the job title, the qualifications needed, and the hiring manager’s name, among others.


```python

# Initialize an empty list to hold all the scraped job data
job_data = []

# Iterate over each job link that we've previously loaded
for job in loaded_list:
    try:
        # Navigate to the job URL
        driver.get(job)

        job_details = {}  # Initialize an empty dictionary to hold details for this job

        # Scraping different fields for job details
        try:
            job_details['job_title'] = driver.find_element_by_class_name("jobs-unified-top-card__job-title").text
        except Exception:
            print("Exception when fetching job title")
            print(traceback.format_exc())

        try:
            job_details['job_metadata'] = driver.find_element_by_class_name("jobs-unified-top-card__primary-description").text
        except Exception:
            print("Exception when fetching job metadata")
            print(traceback.format_exc())

        try:
            job_details['hiring_team_name'] = driver.find_element_by_class_name("jobs-poster__name").text
        except Exception:
            print("Exception when fetching hiring team name")
            print(traceback.format_exc())

        try:
            test = driver.find_element_by_class_name("hirer-card__hirer-information")
            a_element = test.find_element_by_xpath(".//a")
            job_details['hiring_team_link'] = a_element.get_attribute("href")
        except Exception:
            print("Exception when fetching hiring team link")
            print(traceback.format_exc())

        company_metadata = []
        try:
            for i in driver.find_elements_by_class_name("jobs-unified-top-card__job-insight"):
                company_metadata.append(i.text)
            job_details['company_metadata'] = company_metadata
        except Exception:
            print("Exception when fetching company metadata")
            print(traceback.format_exc())

        try:
            job_details['description'] = driver.find_element_by_class_name("jobs-description-content__text").text
        except Exception:
            print("Exception when fetching job description")
            print(traceback.format_exc())

        # Append the collected job details to the master list
        job_data.append(job_details)

    except Exception:
        print("Exception when processing job: ", job)
        print(traceback.format_exc())

```


```python
# Convert the list of job details dictionaries into a DataFrame
df = pd.DataFrame(job_data)

# Add a column for the LinkedIn job links
df['link'] = loaded_list  # We already loaded these job links in a previous step

# Preview the DataFrame to check the data
df.head()

# OPTIONAL: If you've saved the data to a CSV and you're reading it back in
# df = pd.read_csv("final_output.csv", index_col=0)

# Clean up the 'description' column to replace newline characters with a space
df['description'] = df['description'].str.replace(r'\r?\n', ' ', regex=True)
```

## Leveraging Hugging Face's Transformers for Text Analysis
In recent years, Natural Language Processing (NLP) has made significant strides, and much of that has been possible due to the groundbreaking work in transformers. Hugging Face, an AI research organization, has made it easier for developers to access these high-end machine learning models for various NLP tasks through their Transformers library. This open-source library offers a collection of pre-trained models and pipelines to perform tasks such as text summarization, text generation, translation, and more.

Hugging Face's Transformers is particularly helpful in our context of job application automation because it allows us to better understand and interact with job descriptions and requirements. With pre-trained models, we can automatically summarize lengthy job descriptions to quickly grasp essential details or even generate personalized cover letters or application snippets based on job postings.

### Summarization with Hugging Face
The summarization task is invaluable when you're dealing with extensive job descriptions that require a lot of time to read through. With just a few lines of code, you can summarize these descriptions, capturing the most important points. We'll be using "sshleifer/distilbart-cnn-12-6," a model fine-tuned for summarization tasks.

### Text Generation with Hugging Face
Beyond summarization, text generation can offer more creative ways to interact with job data. For example, based on the skills required in the job descriptions, you could generate a paragraph highlighting your matching skills and experiences. For this, we'll use the "meta-llama/Llama-2-7b-chat-hf" model, which specializes in generating human-like conversational text.


```python
# Import the pipeline from the transformers library
from transformers import pipeline

# Initialize the summarization pipeline
get_completion = pipeline("summarization", model="sshleifer/distilbart-cnn-12-6")

# Initialize the text-generation pipeline
pipe = pipeline("text-generation", model="tiiuae/falcon-7b-instruct")

```

Once we have the summarization pipeline set up, the next logical step is to apply this capability to our existing job descriptions. Summarizing each job description can make it easier to quickly understand the essence of each job posting.

### The summarize Function
Let's now define a function called summarize, which will take an input text and return its summarized version. We'll also include exception handling to make sure that the function fails gracefully if it encounters any issues.


```python
def summarize(input):
    try:
        # Using the get_completion pipeline to summarize the input
        output = get_completion(input)
        return output[0]['summary_text']
    except ValueError:
        # Handle the ValueError here if it occurs
        pass

# Applying the summarize function to the 'description' column in our DataFrame
df['description_summary'] = df['description'].apply(summarize)

# Saving the DataFrame with the summarized descriptions
df.to_csv("summarized_description.csv")

```

## Auto-Generating Cover Letters with Text Generation
After summarizing the job descriptions, a key element that remains is creating a cover letter tailored to each job. Writing a unique and compelling cover letter for every job application is a time-consuming process. Wouldn't it be wonderful if we could automate this as well?

In this section, we will define a function that will utilize Hugging Face's text generation capabilities to produce a personalized cover letter for each job posting.

### The generate_cover_letter Function
We will define a function called generate_cover_letter that will take in the row of the DataFrame, which contains summarized job descriptions, job titles, and other metadata. This function will generate a cover letter specific to each job description.

Here's the function code snippet:


```python
def generate_cover_letter(row):
    # Create a prompt incorporating company name, job title, and description summary
    prompt = f"Dear Hiring Team at {row['Company']},\n\nI am excited to apply for the {row['job_title']} position. I was particularly drawn to this role because {row['description_summary']}. Can you please generate a cover letter for me?"

    try:
        # Using the 'pipe' pipeline for text generation based on the prompt
        generated_text = pipe(prompt, max_length=500, do_sample=True, top_k=50)[0]['generated_text']
        
        # Extracting the generated cover letter from the generated_text
        start = generated_text.find("Dear Hiring Team")
        if start != -1:
            generated_cover_letter = generated_text[start:]
            return generated_cover_letter

    except Exception as e:
        print(f"An error occurred: {e}")
        return None

# Applying the generate_cover_letter function to each row in the DataFrame
df['generated_cover_letter'] = df.apply(generate_cover_letter, axis=1)

# Saving the DataFrame with generated cover letters
df.to_csv("with_generated_cover_letters.csv")

```

## Updating the generation part with ollama instead

Since I wrote this article, ollama came out which if I'm being honest is porbably the best tool to run llm models as it supports calling the function straight from python or terminal.

Let's start by re reading the [last csv](summarized_description.csv)


```python
import ollama 
import pandas as pd

df = pd.read_csv("summarized_description.csv", index_col=0)


def compose_email(hiring_team_name, job_title, job_metadata, hiring_team_link, company_metadata,description, link, description_summary):
    """
    Generates a personalized email for a job application using provided details.

    Parameters:
    - hiring_team_name: The name of the person in the hiring team.
    - job_title: The title of the job.
    - job_metadata: Additional details about the job.
    - hiring_team_link: A link to learn more about the hiring team or the person.
    - company_metadata: Information about the company.
    - link: A link to the job posting.
    - description_summary: A summary of the job description, emphasizing key responsibilities and qualifications.

    Returns:
    A string containing the personalized email.
    """
    email_body = f"""
Dear {hiring_team_name},

I am reaching out to express my genuine interest in the {job_title} position as advertised on {link}. With a solid background in {job_metadata}, I am enthusiastic about the opportunity to contribute to {company_metadata}, a company I greatly admire for its [insert reason based on company_metadata].

The role's focus on {description_summary} aligns perfectly with my professional skills and personal interests. I am confident that my experience in [mention relevant experience] makes me a strong candidate for this position. I am particularly excited about the chance to [mention a specific project or responsibility mentioned in the job description].

Here’s why I believe I am a good fit for the role:
- [Briefly highlight key qualifications and experiences relevant to the job description_summary]

I am very much looking forward to the possibility of discussing this exciting opportunity with you further. Please let me know if you need any more information or if there's a convenient time for us to discuss how I can contribute to your team.

Thank you for considering my application.

Best regards,
[Your Name]
"""

    return email_body.strip()

```


```python
# Bring back first row for testing
row = df.iloc[0]
row

```




    job_title                                            Investment Engineer
    job_metadata           Bridgewater Associates · Westport, CT (On-site...
    hiring_team_name                                         Jason Koulouras
    hiring_team_link              https://www.linkedin.com/in/jasonkoulouras
    description            About the job About Bridgewater  Bridgewater A...
    company_metadata                                                     NaN
    link                   https://www.linkedin.com/jobs/view/3659222868/...
    description_summary     Investment Engineer's mission is to understan...
    Name: 0, dtype: object




```python
message = compose_email(**row)
message

model_input = {
    'role': 'system',  # or 'user', depending on the context
    'content': message
}
```


```python
response = ollama.chat(model='llama2', messages=[model_input])
print(response['message']['content'])

## Create a class or a function  to loop over the dataframe and send emails to the hiring team if you would like to
```

    Dear Jason Koulouras,
    
    I am writing to express my strong interest in the Investment Engineer position at Bridgewater Associates. I was drawn to this opportunity due to the company's reputation as a leader in the financial industry and its commitment to innovation and excellence. As an experienced investment professional with a solid background in [relevant field], I am confident that my skills and experience align perfectly with the requirements of this role.
    
    From my research on Bridgewater Associates, I understand that the Investment Engineer's primary responsibility is to design and build the algorithms used to generate the company's views on markets and economies, as well as translate those views into portfolios and trade them. As someone who shares the company's passion for understanding how these systems work and using that knowledge to inform investment decisions, I am excited about the opportunity to contribute to this mission.
    
    In my current role at [current company], I have gained valuable experience in [relevant skills or experiences]. These skills, combined with my ability to work well in a team and communicate complex ideas effectively, make me a strong candidate for this position. I am particularly enthusiastic about the opportunity to work on [specific project or responsibility mentioned in the job description] and contribute my expertise to help drive Bridgewater's success.
    
    Thank you for considering my application. I would be thrilled to discuss how I can contribute to your team and help drive the company's continued growth and innovation. Please let me know if there is any additional information you need or if you would like to schedule a time to speak further.
    
    Sincerely,
    [Your Name]


#### NOTE:
This is not perfect, and you should not automate this application part. The whole idea of this project is to automate the data collection part of finding your perfect job; the application should always have your personal touch, as it will be the only thing that will stand out from the 1000 resumes that are being flooded 

## Conclusion

You can still do more with the LLM part, filter out jobs you don't want, or get involved in creating more specialized emails. The sky (or your processing power) is the limit.


Automation in job applications, mainly through data scraping and machine learning, can significantly streamline the hiring process for job seekers and recruiters. This tutorial demonstrates how one can automatically collect job postings from LinkedIn and then use machine learning models, specifically from Hugging Face's Transformers library, to generate personalized cover letters.

We've walked through how to set up a web scraper with Selenium, fetch relevant job details, and utilize the NLP power of Hugging Face to generate content dynamically.

However, I would like to point out that the applicant should review and fine-tune the generated cover letters to ensure they align with the individual's unique skills, experience, and aspirations. Automation tools can accelerate the process but only partially replace the human touch.

Lastly, please ensure you comply with the terms and conditions or guidelines set by any third-party services you use. This tutorial is strictly educational and should not be considered an encouragement to violate such terms.

Happy job hunting, and may your automation efforts land your dream job!

That wraps up our article. I hope this tutorial is informative and useful. Thank you for reading!
