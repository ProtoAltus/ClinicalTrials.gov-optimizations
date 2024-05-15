# Code for API generation
# Module 1 : Setting basic foundations and parameters

import csv
import requests    # Install these, if absent for the program to run successfully
import time

# CSV functions
def clear_csv():
    with open(f'{csv_file_name}.csv', 'w', newline='') as file:
        writer = csv.writer(file)
        writer.writerow(["Biomarker", "Organizer", "Phase 1", "Phase 2", "Phase 3", "Phase 4", "Total Phase 1", "Total Phase 2", "Total Phase 3", "Total Phase 4"])   # List of the rows to be created

def read_whitelist(filename):
    with open(filename, 'r') as file:
        whitelist = [line.strip() for line in file if line.strip()]
    return whitelist

def replace_special_characters(text):
    return text.replace(' ', '+').replace('&', '%26').replace('#','%23').replace('™','%99').replace('®','%AE').replace(':','%3A').replace('/','%2F').replace('.','%2E')

def return_special_characters(text):
    return text.replace('+', ' ').replace('%26', '&').replace('%23','#').replace('%99','™').replace('%AE','®').replace('%3A',':').replace('%2F','/').replace('%2E','.')

biomarker_whitelist = read_whitelist('biomarker_whitelist.txt') # Files containing all values to be read
organizer_whitelist = read_whitelist('organizer_whitelist.txt') 
skip_organizer = ['']   # Skipping specific companies entered in the string
skipover_biomarker = ['']   # Skipping specific biomarkers entered in the string
csv_file_name = 'INSERT_FILE_NAME' # Type the required file name here


# Module 2 :Starting loops and setting counters

company_phases = {}
valid_values_counter = 0  

# Processes only these biomarkers in form, " x, y, z, .... "
priority_biomarker_string = ""  # Set to 'None' for all biomarkers
priority_biomarkers = [bm.strip() for bm in priority_biomarker_string.split(',')] if priority_biomarker_string else None

if priority_biomarkers is not None:
    # If any present, processes only those biomarkers
    biomarkers_to_process = priority_biomarkers
else:
    # If none are, processes the entire list
    biomarkers_to_process = biomarker_whitelist

if biomarkers_to_process and organizer_whitelist:
    clear_csv()  # Clears the CSV file whenever you execute

    for biomarker in biomarkers_to_process:
        sanitized_biomarker = replace_special_characters(biomarker)
        company_phases[sanitized_biomarker] = {}
        if biomarker in skipover_biomarker:
            print(f"Already finished biomarkers: {biomarker}")
            continue  # Skips to the next biomarker if in skipover_biomarker['']

        total_phases_for_biomarker = {"Total Phase 1": 0, "Total Phase 2": 0, "Total Phase 3": 0, "Total Phase 4": 0} # Sets a total counter for every specific marker

        for organizer in organizer_whitelist:
            if organizer in skip_organizer:
                print(f"Skipping Organizer {organizer}...")
                continue
            
            # Cleans up company name to fit url
            sanitized_organizer = replace_special_characters(organizer)
            url_template = f'https://clinicaltrials.gov/api/v2/studies?format=json&query.term={sanitized_biomarker}&query.lead={sanitized_organizer}'
            url = url_template
            response = requests.get(url_template)

            if response.status_code == 200:  # 200 is the status code for successful execution.
                data = response.json()
                studies = data.get('studies', [])
                phase_counters = {"Phase 1": 0, "Phase 2": 0, "Phase 3": 0, "Phase 4": 0}

# Module 3 :Looping the printing of data into the csv file        
                
                # Main loop for Phase value checking in API response in array format.
                if studies:
                    for study in studies:
                        protocol_section = study.get('protocolSection', {})
                        design_module = protocol_section.get('designModule', {})
                        phases = design_module.get('phases', [])

                        if not phases:
                            print(f"No Phases Found...")
                        else:

                            # Conditions to check if Phases are present
                            print(f"          Link obtained: {return_special_characters(biomarker)}/ Organizer: {return_special_characters(organizer)} \n")
                            if "PHASE1" in phases:
                                phase_counters["Phase 1"] += 1
                                total_phases_for_biomarker["Total Phase 1"] += 1
                            if "PHASE2" in phases:
                                phase_counters["Phase 2"] += 1
                                total_phases_for_biomarker["Total Phase 2"] += 1
                            if "PHASE3" in phases:
                                phase_counters["Phase 3"] += 1
                                total_phases_for_biomarker["Total Phase 3"] += 1
                            if "PHASE4" in phases:
                                phase_counters["Phase 4"] += 1
                                total_phases_for_biomarker["Total Phase 4"] += 1
                            total_phases_biomarker_cumulative = phase_counters["Phase 4"] + phase_counters["Phase 4"] + phase_counters["Phase 4"] + phase_counters["Phase 4"]
                
                     
                    # Prints the data
                    print(f"          All studies processed for Biomarker: {return_special_characters(biomarker)}, Organizer: {return_special_characters(organizer)}")
                    print(f"          \n Valid Link(s)! Phase Counters Biomarker: {return_special_characters(biomarker)} and Organizer: {return_special_characters(organizer)} - {phase_counters} \n",url) # Use to check if the generated URL has values or not, for manual checking
                    time.sleep(0.25)

                    # Appends data to CSV file named in the variable above
                    with open(f'{csv_file_name}.csv', 'a', newline='') as file:
                        writer = csv.writer(file)
                        writer.writerow([return_special_characters(biomarker), return_special_characters(organizer),
                                         phase_counters["Phase 1"], phase_counters["Phase 2"],
                                         phase_counters["Phase 3"], phase_counters["Phase 4"],
                                         total_phases_for_biomarker["Total Phase 1"], total_phases_for_biomarker["Total Phase 2"],
                                         total_phases_for_biomarker["Total Phase 3"], total_phases_for_biomarker["Total Phase 4"]])
                    valid_values_counter += 1  # Increments counter value for sleep checking

                    if valid_values_counter == 25:
                        print("Sleeping for 25 values...")
                        time.sleep(30)  # Pauses for 30 seconds, to modify you can change the value to however long you need it to be
                        valid_values_counter = 0  # Resets the counter after sleeping
                else:
                    print(f"    No studies found for Biomarker: {return_special_characters(biomarker)}, Organizer: {return_special_characters(organizer)}")

        # Prints total phases for biomarker loop
        print(f"\nTotal Phases for Biomarker: {return_special_characters(biomarker)} - {total_phases_for_biomarker} \n Total phases = ", total_phases_biomarker_cumulative)
        # Resets total phases for biomarker loop
        total_phases_for_biomarker = {"Total Phase 1": 0, "Total Phase 2": 0, "Total Phase 3": 0, "Total Phase 4": 0}

    # Prints after all data is appended
    print("\nData has been appended to the CSV file.")

# When no whitelist is set
else:
    print("Whitelist(s) not found, check again")
