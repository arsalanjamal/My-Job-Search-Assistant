import streamlit as st
from selenium import webdriver
from selenium.webdriver.common.by import By
import pandas as pd
import time
from io import StringIO

# Setup Streamlit Page
st.set_page_config(page_title="Job Search Assistant", layout="wide")

# Title and description
st.title("Job Search Assistant")
st.markdown("""
This assistant helps you find job listings from Indeed based on your query. You can enter a job title or skill, and it will show relevant jobs with title, company, salary (if available), and a link to apply. You can also filter by location and experience level.
""")

# Input fields for user query and filters
job_title = st.text_input("Enter the job title or skill:", "")
location = st.text_input("Enter location (e.g., New York, Remote):", "")
experience = st.selectbox("Experience Level:", ["Any", "Entry-level", "Mid-level", "Senior-level"])

# Function to scrape job listings from Indeed
def scrape_jobs(query, location, experience):
    base_url = "https://www.indeed.com/jobs"
    search_url = f"{base_url}?q={query}&l={location}"
    
    # Setup WebDriver (using Chrome)
    options = webdriver.ChromeOptions()
    options.add_argument('--headless')  # Run headless for better performance
    driver = webdriver.Chrome(options=options)
    driver.get(search_url)

    # Wait for page to load
    time.sleep(3)

    # Filter by experience level if applicable
    if experience != "Any":
        driver.find_element(By.LINK_TEXT, experience).click()
        time.sleep(2)

    jobs = []

    # Loop through job listings and extract relevant information
    job_elements = driver.find_elements(By.CSS_SELECTOR, "div.job_seen_beacon")
    
    for job in job_elements:
        title = job.find_element(By.CSS_SELECTOR, "h2.jobTitle").text
        company = job.find_element(By.CSS_SELECTOR, "span.companyName").text
        salary = job.find_element(By.CSS_SELECTOR, "div.salary-snippet").text if job.find_elements(By.CSS_SELECTOR, "div.salary-snippet") else "Not specified"
        link = job.find_element(By.TAG_NAME, 'a').get_attribute('href')

        jobs.append({
            'Job Title': title,
            'Company': company,
            'Salary': salary,
            'Link to Apply': link
        })
    
    driver.quit()  # Close the driver after scraping is done
    return jobs

# Function to export job results to CSV
def export_to_csv(jobs):
    df = pd.DataFrame(jobs)
    csv_data = df.to_csv(index=False)
    return StringIO(csv_data)

# Main function to handle app logic
def main():
    # Show a loading spinner while scraping jobs
    if st.button("Search Jobs"):
        if job_title:
            with st.spinner("Searching for jobs..."):
                jobs = scrape_jobs(job_title, location, experience)
                
                if jobs:
                    st.success(f"Found {len(jobs)} job listings!")
                    for job in jobs:
                        st.write(f"**Job Title**: {job['Job Title']}")
                        st.write(f"**Company**: {job['Company']}")
                        st.write(f"**Salary**: {job['Salary']}")
                        st.write(f"[Apply Here]({job['Link to Apply']})\n")
                    
                    # Option to download the job listings as CSV
                    csv_data = export_to_csv(jobs)
                    st.download_button(label="Download Job Listings as CSV", data=csv_data, file_name="job_listings.csv")
                else:
                    st.warning("No jobs found matching your criteria.")
        else:
            st.error("Please enter a job title or skill.")

if __name__ == "__main__":
    main()
