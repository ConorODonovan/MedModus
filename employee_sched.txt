# MedModus Junior Analyst Programmer- Optimization Coding Exercise
# Conor O'Donovan
# Email: conor.m.odonovan@gmail.com
# Phone: 0894844175

#########
# NOTES #
#########
# Please ensure that the Nurses.csv file is in the same folder as this .py file so it can be read
# I did not answer Question 3, as I did not understand it. The model would evenly distribute all
# shifts, regardless of the day of the week, so I did not know how anything could be applied here


# Import the libraries
from ortools.sat.python import cp_model
import pandas as pd
import csv # For question 1 - reading the data from the CSV file

# Read Nurses.csv file and put into list, each list element containing nurse number and team
with open('Nurses.csv', newline='') as f:
    reader = csv.reader(f)
    data = list(reader)
    nurse_list = data[1:] # Remove header

# Create the data


##############
# QUESTION 2 #
##############
num_nurses = len(nurse_list) # Question 2 - update num_nurses to work with imported data - num_nurses now correctly equals 10 as per the CSV file

num_shifts = 3


##############
# QUESTION 1 #
##############
num_days = 56 # Question 1 - change number of days from 4 to 56

all_nurses = range(num_nurses) # As per line 16, this is now range[0,10] as per imported CSV file

all_shifts = range(num_shifts)
all_days = range(num_days)

# Create the model
model = cp_model.CpModel()

# Create the variables for the problem
shifts = {}
for d in all_days:
    for n in all_nurses:
        for s in all_shifts:
            shifts[(n, d, s)] = model.NewBoolVar('shift_n%sd%is%i' % (n, d, s))

# Each shift is assigned to a single nurse per day
for d in all_days:
    for s in all_shifts:
        model.Add(sum(shifts[(n, d, s)] for n in all_nurses) == 1)

    # Each nurse works at most one shift per day
for d in all_days:
    for n in all_nurses:
        model.Add(sum(shifts[(n, d, s)] for s in all_shifts) <= 1)


##############
# QUESTION 5 #
##############
# Question 5 - for shifts 0 and 1, ensure that a nurse does not work either of these shifts for two days consecutively
for d in range(num_days - 2):
    for n in all_nurses:
        model.Add(sum(shifts[(n, d, s)] for s in range(2)) + sum(shifts[(n, d + 1, s)] for s in range(2)) + sum(shifts[(n, d + 2, s)] for s in range(2)) <= 2)


##############
# QUESTION 6 #
##############
# Question 6 for shift 2, ensure that the nurse working shift 2 on a Monday also works shift 2 on Tuesday and Wednesday
for d in range(num_days):
    if d == 0 or d % 7 == 0:
        for n in all_nurses:
            model.Add(sum(shifts[(n, d, s)] for s in all_shifts if s == 2 for d in all_days if d == 0 or d % 7 == 0) + sum(shifts[(n, d, s)] for s in all_shifts if s == 2 for d in all_days if d == 1 or (d - 1) % 7 == 0) + sum(shifts[(n, d, s)] for s in all_shifts if s == 2 for d in all_days if d == 2 or (d - 2) % 7 == 0) <= 3)


##############
# QUESTION 7 #
##############
# Question 7 - ensure each nurse works no more than 4 shifts a week (Mon - Sun)
for n in all_nurses:
    for d in all_days:
        if d == 0 or d % 7 == 0:
            model.Add(sum(shifts[(n, d, s)] for s in all_shifts) + sum(shifts[(n, d + 1, s)] for s in all_shifts) + sum(shifts[(n, d + 2, s)] for s in all_shifts) + sum(shifts[(n, d + 3, s)] for s in all_shifts) + sum(shifts[(n, d + 4, s)] for s in all_shifts) + sum(shifts[(n, d + 5, s)] for s in all_shifts) + sum(shifts[(n, d + 6, s)] for s in all_shifts) <= 4)
    

##############
# QUESTION 8 #
##############
# Question 8 - ensure nurses from Team A do not work shift 0
for n in all_nurses:
    for d in all_days:
        model.Add(sum(shifts[(n, d, s)] for s in all_shifts if s == 0 for n in all_nurses if nurse_list[n][1] == 'A') == 0)
    

##############
# QUESTION 9 #
##############
# Question 9 - ensure no more than 1 nurse from each team is working each day
for d in all_days:
    model.Add(sum(shifts[(n, d, s)] for s in all_shifts for n in all_nurses if nurse_list[n][1] == 'A') + sum(shifts[(n, d, s)] for s in all_shifts for n in all_nurses if nurse_list[n][1] == 'B') + sum(shifts[(n, d, s)] for s in all_shifts for n in all_nurses if nurse_list[n][1] == 'C') >= 1)


# Assign shifts evenly
# Try to distribute the shifts evenly, so that each nurse works
# min_shifts_per_nurse shifts. If this is not possible, because the total
# number of shifts is not divisible by the number of nurses, some nurses will
# be assigned one more shift.
min_shifts_per_nurse = (num_shifts * num_days) // num_nurses
if (num_shifts * num_days) % num_nurses == 0:
    max_shifts_per_nurse = min_shifts_per_nurse
else:
    max_shifts_per_nurse = min_shifts_per_nurse + 1

for n in all_nurses:
    num_shifts_worked = sum(shifts[(n, d, s)] for d in all_days for s in all_shifts)
    model.Add(num_shifts_worked >= min_shifts_per_nurse)
    model.Add(num_shifts_worked <= max_shifts_per_nurse)

solver = cp_model.CpSolver()
status = solver.Solve(model)

if status == cp_model.OPTIMAL:
    rota_dict = {(n, d, s) for (n, d, s) in shifts if solver.Value(shifts[(n, d, s)]) == 1}
    Solution = pd.DataFrame(rota_dict, columns=['Nurse', 'Day', 'Shift']).sort_values(by=['Day'])
    Rota = Solution.pivot(index='Day', columns='Shift', values='Nurse')
    print(Rota)
else:
    print('ERROR: no solution')


##############
# QUESTION 4 #
##############

# Question 4 - showing how many total and weekend shifts each nurse has been given
rota_list = Rota.values.tolist()

# Total shifts per nurse
nurse0 = 0
nurse1 = 0
nurse2 = 0
nurse3 = 0
nurse4 = 0
nurse5 = 0
nurse6 = 0
nurse7 = 0
nurse8 = 0
nurse9 = 0

# Count number of shifts per nurse across entire rota
for i in rota_list:
    for j in i:
        if j == 0:
            nurse0 += 1
        elif j == 1:
            nurse1 += 1
        elif j == 2:
            nurse2 += 1
        elif j == 3:
            nurse3 += 1
        elif j == 4:
            nurse4 += 1
        elif j == 5:
            nurse5 += 1
        elif j == 6:
            nurse6 += 1
        elif j == 7:
            nurse7 += 1
        elif j == 8:
            nurse8 += 1
        elif j == 9:
            nurse9 += 1

# Printing results nicely formatted
# Total shifts per nurse
print("\n")
print("==============================================================")
print ("Total shifts for each nurse:")
print("==============================================================")
print("Nurse 1: ", nurse0)
print("Nurse 2: ", nurse1)
print("Nurse 3: ", nurse2)
print("Nurse 4: ", nurse3)
print("Nurse 5: ", nurse4)
print("Nurse 6: ", nurse5)
print("Nurse 7: ", nurse6)
print("Nurse 8: ", nurse7)
print("Nurse 9: ", nurse8)
print("Nurse 10: ", nurse9)

# Weekend shifts per nurse


nurse0 = 0
nurse1 = 0
nurse2 = 0
nurse3 = 0
nurse4 = 0
nurse5 = 0
nurse6 = 0
nurse7 = 0
nurse8 = 0
nurse9 = 0

day = 0

for i in rota_list:
    if (day % 5 == 0 or day % 6 == 0) and day != 0:
        for j in i:
            if j == 0:
                nurse0 += 1
            elif j == 1:
                nurse1 += 1
            elif j == 2:
                nurse2 += 1
            elif j == 3:
                nurse3 += 1
            elif j == 4:
                nurse4 += 1
            elif j == 5:
                nurse5 += 1
            elif j == 6:
                nurse6 += 1
            elif j == 7:
                nurse7 += 1
            elif j == 8:
                nurse8 += 1
            elif j == 9:
                nurse9 += 1

    day += 1

print("\n")
print("==============================================================")
print ("Total weekend shifts for each nurse:")
print("==============================================================")
print("Nurse 1: ", nurse0)
print("Nurse 2: ", nurse1)
print("Nurse 3: ", nurse2)
print("Nurse 4: ", nurse3)
print("Nurse 5: ", nurse4)
print("Nurse 6: ", nurse5)
print("Nurse 7: ", nurse6)
print("Nurse 8: ", nurse7)
print("Nurse 9: ", nurse8)
print("Nurse 10: ", nurse9)
print("\n")