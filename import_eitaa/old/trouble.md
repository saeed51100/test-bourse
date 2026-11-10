I am a developer working with Python on Ubuntu. 
I would like you to help me troubleshoot Section 9 of a Python/Selenium program designed to automatically add contacts to Eitaa (an Iranian messaging app).

### Development Environment 
- Operating System: Ubuntu 
- Programming Language: Python 
- Python Environment: Conda (environment.yml) 
- Programming Environment: Jupyter Notebook (.ipynb) 
- IDE: PyCharm 

### Project Overview 
I have written a Python program that is intended to add approximately 5,000 contacts to Eitaa Messenger, an Iranian messaging platform.

* The manual process for adding a contact in "Eitaa Web" is as follows:
a. Open "Eitaa Web" in the browser and make sure the user is already logged in. 
b. Click the hamburger menu button to open the main navigation menu. 
c. From the menu, click Contacts.
d. The Contacts page opens and displays the existing contact list. Click the "+" (Add Contact) button to open the "Add Contact" form. 
e. The "Add Contact" form contains the following fields:
- First Name (Required)
- Last Name (Optional )
- Phone Number Required; +98 has already been entered and is present in the form. 
f. Enter the contact information into the appropriate fields. 
g. Click the "Add" button. 
h. After clicking "Add" Eitaa Web should save the contact, close the "Add Contact"
 form, and return to the Contacts page. The newly added contact should then appear in the contact list. 
i. For the Selenium automation, the objective is to reproduce this same sequence one contact at a time: Open Contacts → Click "+" → Fill First Name → Fill Last Name (if available) → Fill Phone Number → Click "Add" → Verify that the contact was successfully added.

* The ipynb file contains the following sections and runs without errors:
1- Database Design in phpMyAdmin → SQL 
2- import_runs Table → SQL 
3- Phone Number Normalization 
4- Import Excel Data into the Database 
5- Chrome Session Persistence 
6- Close Selenium and remove temporary Chrome profile 
7- Selenium Selectors 
8- Reliable Waits 
9- Actual Eitaa Contact-Adding Operation 
10- Fetching the Next Contact 
11- Complete Processing Loop

* The problem:
The issue is that when the program runs, contacts are not added to Eitaa, and our goal—automatically adding contacts—fails.

* My request: 
- I have already troubleshot sections 1 through 8, and they are working correctly. 
- I would like you to troubleshoot section 9 step-by-step and provide me with the corrected, complete version of it. 
I am including the code for section 9 below.

* Important Note 1:
After extensive testing, it was determined that Selenium and the selectors were not the root cause of the problem:
- The hamburger/menu button was also successfully clicked and opened. ✅
- Open Contacts ✅
- Open "Add Contact" ✅
- First Name ✅
- Phone ✅
- "ADD" button becomes active ✅
- Click "ADD" ✅
- A UI test was performed using a number definitely registered on Eitaa, and the contact was successfully added to the contacts list ✅
- In application test, Eitaa responds that the number is not registered ❌
- The popup does not close; therefore, the current `invisibility_of_element_located` expectation is unsuitable for the error state.

* Important Note 2 : For troubleshooting Section 9, I do not want any additional explanations or justifications for your actions. Instead, I want a rapid troubleshooting process where you provide code snippets, I execute them and send you the output, and we ultimately arrive at the corrected code for Section 9.

*  Section 9:
# ============================================================
# 9- Actual Eitaa Contact
# ============================================================

def add_contact(driver, full_name, phone_normalized):
    """
    Add exactly one Eitaa contact.

    Full name intentionally goes entirely into
    the first-name field.
    """

    # If you are not already in Contacts, sign in to Contacts.
    contacts_container = driver.find_element(
        By.CSS_SELECTOR,
        "#contacts-container"
    )

    if not contacts_container.is_displayed():
        wait_clickable(driver, Selectors.HAMBURGER_MENU).click()
        wait_clickable(driver, Selectors.CONTACTS_BUTTON).click()

    # Add Contact
    add_btn = WebDriverWait(driver, WAIT_SECONDS).until(
        EC.element_to_be_clickable(
            (By.CSS_SELECTOR, "#contacts-container button.tgico-add")
        )
    )
    add_btn.click()

    # Form
    first_name = wait_visible(driver, Selectors.FIRST_NAME)
    first_name.click()
    first_name.send_keys(full_name)

    phone = wait_visible(driver, Selectors.PHONE)
    phone.click()
    phone.send_keys(phone_normalized)

    # Save
    save = wait_clickable(driver, Selectors.SAVE_BUTTON)
    save.click()

    # Wait until the popup closes.
    WebDriverWait(driver, WAIT_SECONDS).until(
        EC.invisibility_of_element_located(
            (By.CSS_SELECTOR, ".popup-create-contact.active")
        )
    )

    return True


