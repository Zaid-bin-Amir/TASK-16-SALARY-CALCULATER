SALARY_RULES = {
    "allowances": {
        "hra_percent": 20,
        "da_percent": 10,
        "medical_allowance": 1500,
        "transport_allowance": 1000,
    },
    "deductions": {
        "pf_percent": 8,
        "professional_tax": 200,
    },
    "performance_bonus_percent": {
        "A": 15,
        "B": 10,
        "C": 5,
        "D": 0,
    },
    "tax_slabs": [
        (25000, 0),
        (50000, 5),
        (100000, 10),
        (float("inf"), 20),
    ],
    "overtime_rate_per_hour": 300,
}

EMPLOYEES = [
    {"id": 101, "name": "Ali Khan", "basic": 40000, "rating": "A", "overtime_hours": 5},
    {"id": 102, "name": "Sara Ahmed", "basic": 65000, "rating": "B", "overtime_hours": 0},
    {"id": 103, "name": "Usman Raza", "basic": 22000, "rating": "C", "overtime_hours": 12},
    {"id": 104, "name": "Hina Malik", "basic": 90000, "rating": "A", "overtime_hours": 2},
]


def validate_number(value, field_name, minimum=0):
    try:
        number = float(value)
    except (TypeError, ValueError):
        raise ValueError(f"{field_name} must be a number, got '{value}'")
    if number < minimum:
        raise ValueError(f"{field_name} cannot be less than {minimum}")
    return number


def validate_employee(employee, rules):
    required = ("id", "name", "basic", "rating", "overtime_hours")
    missing = [key for key in required if key not in employee]
    if missing:
        raise ValueError(f"Missing fields: {', '.join(missing)}")
    basic = validate_number(employee["basic"], "Basic salary", minimum=1)
    overtime = validate_number(employee["overtime_hours"], "Overtime hours")
    rating = str(employee["rating"]).upper()
    if rating not in rules["performance_bonus_percent"]:
        valid = ", ".join(rules["performance_bonus_percent"])
        raise ValueError(f"Rating must be one of: {valid}")
    return basic, overtime, rating


def percent_of(amount, percent):
    return round(amount * percent / 100, 2)


def calculate_gross(basic, rating, overtime_hours, rules):
    allowances = rules["allowances"]
    components = {
        "Basic": basic,
        "HRA": percent_of(basic, allowances["hra_percent"]),
        "DA": percent_of(basic, allowances["da_percent"]),
        "Medical": allowances["medical_allowance"],
        "Transport": allowances["transport_allowance"],
        "Performance Bonus": percent_of(basic, rules["performance_bonus_percent"][rating]),
        "Overtime": round(overtime_hours * rules["overtime_rate_per_hour"], 2),
    }
    return components, round(sum(components.values()), 2)


def calculate_income_tax(gross, slabs):
    tax = 0.0
    lower = 0
    for upper, rate in slabs:
        if gross > lower:
            taxable = min(gross, upper) - lower
            tax += taxable * rate / 100
        lower = upper
    return round(tax, 2)


def calculate_deductions(basic, gross, rules):
    deductions_rules = rules["deductions"]
    deductions = {
        "Provident Fund": percent_of(basic, deductions_rules["pf_percent"]),
        "Professional Tax": deductions_rules["professional_tax"],
        "Income Tax": calculate_income_tax(gross, rules["tax_slabs"]),
    }
    return deductions, round(sum(deductions.values()), 2)


def calculate_salary(employee, rules=SALARY_RULES):
    basic, overtime, rating = validate_employee(employee, rules)
    earnings, gross = calculate_gross(basic, rating, overtime, rules)
    deductions, total_deductions = calculate_deductions(basic, gross, rules)
    return {
        "id": employee["id"],
        "name": employee["name"],
        "rating": rating,
        "earnings": earnings,
        "gross": gross,
        "deductions": deductions,
        "total_deductions": total_deductions,
        "net": round(gross - total_deductions, 2),
    }


def print_breakdown(result):
    line = "=" * 46
    print(line)
    print(f"Employee : {result['name']} (ID: {result['id']})")
    print(f"Rating   : {result['rating']}")
    print(line)
    print("EARNINGS")
    for label, amount in result["earnings"].items():
        print(f"  {label:<20}{amount:>18,.2f}")
    print(f"  {'Gross Salary':<20}{result['gross']:>18,.2f}")
    print("DEDUCTIONS")
    for label, amount in result["deductions"].items():
        print(f"  {label:<20}{amount:>18,.2f}")
    print(f"  {'Total Deductions':<20}{result['total_deductions']:>18,.2f}")
    print(line)
    print(f"  {'NET SALARY':<20}{result['net']:>18,.2f}")
    print(line)
    print()


def print_summary(results):
    print("PAYROLL SUMMARY")
    print(f"{'ID':<6}{'Name':<14}{'Gross':>12}{'Deductions':>14}{'Net':>12}")
    print("-" * 58)
    for r in results:
        print(f"{r['id']:<6}{r['name']:<14}{r['gross']:>12,.2f}{r['total_deductions']:>14,.2f}{r['net']:>12,.2f}")
    total_net = sum(r["net"] for r in results)
    print("-" * 58)
    print(f"{'Total net payout':<46}{total_net:>12,.2f}")
    print()


def run_payroll(employees, rules=SALARY_RULES):
    results = []
    for employee in employees:
        try:
            results.append(calculate_salary(employee, rules))
        except ValueError as error:
            print(f"Skipped {employee.get('name', 'Unknown')}: {error}\n")
    return results


def read_employee_from_user():
    return {
        "name": input("Employee name: ").strip(),
        "id": input("Employee ID: ").strip(),
        "basic": input("Basic salary: ").strip(),
        "rating": input("Performance rating (A/B/C/D): ").strip(),
        "overtime_hours": input("Overtime hours: ").strip(),
    }


def main():
    results = run_payroll(EMPLOYEES)
    for result in results:
        print_breakdown(result)
    print_summary(results)

    choice = input("Add a new employee manually? (y/n): ").strip().lower()
    if choice == "y":
        employee = read_employee_from_user()
        try:
            print_breakdown(calculate_salary(employee))
        except ValueError as error:
            print(f"Invalid input: {error}")


if __name__ == "__main__":
    main()
