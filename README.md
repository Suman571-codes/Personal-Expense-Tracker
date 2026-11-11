#/*
 * ============================================================
 * PERSONAL EXPENSE TRACKER - OOP Project in C++
 * ------------------------------------------------------------
 * Name        : Suman
 * UID         : 25BCD10173
 * Branch      : BCA (Data Science)   Section : 25BCD3 (B)
 * Institute   : Chandigarh University
 * Submitted to: Abhinay Sood
 * ------------------------------------------------------------
 * UNIQUE FEATURE: "No-Spend Streak Challenge"
 * The user starts a challenge (e.g. 14 days) and the program
 * tracks how many consecutive days had ZERO expenses, the best
 * streak, estimated money saved, a monthly calendar and badges.
 * ------------------------------------------------------------
 * Compile : g++ -std=c++11 ExpenseTracker.cpp -o tracker
 * Run     : ./tracker      (Windows: tracker.exe)
 * ============================================================
 */
#include <iostream>
#include <fstream>
#include <sstream>
#include <vector>
#include <map>
#include <string>
#include <iomanip>
#include <ctime>
#include <cstdlib>
#include <limits>
#include <algorithm>
using namespace std;
 
/* ---------- File names used for permanent storage ---------- */
const string DATA_FILE = "transactions.txt";
const string BUDGET_FILE = "budget.txt";
const string CHALLENGE_FILE = "challenge.txt";
 
/* ====================== CLASS: Date ======================== */
class Date {
private:
    int day, month, year;
public:
    Date(int d = 1, int m = 1, int y = 2000) : day(d), month(m), year(y) {}
 
    static Date today() {
        time_t t = time(0);
        tm* lt = localtime(&t);
        return Date(lt->tm_mday, lt->tm_mon + 1, lt->tm_year + 1900);
    }
    static bool isLeap(int y) { return (y % 4 == 0 && y % 100 != 0) || (y % 400 == 0); }
    static int daysInMonth(int m, int y) {
        static const int dm[] = {31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31};
        if (m == 2 && isLeap(y)) return 29;
        return dm[m - 1];
    }
    bool isValid() const {
        return year >= 1970 && year <= 2100 && month >= 1 && month <= 12 &&
               day >= 1 && day <= daysInMonth(month, year);
    }
    int getDay() const { return day; }
    int getMonth() const { return month; }
    int getYear() const { return year; }
 
    /* days since 1970-01-01 (lets us do date arithmetic easily) */
    long toDays() const {
        int yy = year - (month <= 2 ? 1 : 0);
        long era = (yy >= 0 ? yy : yy - 399) / 400;
        long yoe = yy - era * 400;
        long doy = (153 * (month + (month > 2 ? -3 : 9)) + 2) / 5 + day - 1;
        long doe = yoe * 365 + yoe / 4 - yoe / 100 + doy;
        return era * 146097 + doe - 719468;
    }
    static Date fromDays(long z) {
        z += 719468;
        long era = (z >= 0 ? z : z - 146096) / 146097;
        long doe = z - era * 146097;
        long yoe = (doe - doe / 1460 + doe / 36524 - doe / 146096) / 365;
        long yy = yoe + era * 400;
        long doy = doe - (365 * yoe + yoe / 4 - yoe / 100);
        long mp = (5 * doy + 2) / 153;
        long dd = doy - (153 * mp + 2) / 5 + 1;
        long mm = mp < 10 ? mp + 3 : mp - 9;
        return Date((int)dd, (int)mm, (int)(yy + (mm <= 2 ? 1 : 0)));
    }
    string toString() const {
        ostringstream os;
        os << setfill('0') << setw(2) << day << "-" << setw(2) << month << "-" << year;
        return os.str();
    }
 
    /* operator overloading */
    bool operator==(const Date& o) const { return toDays() == o.toDays(); }
    bool operator<(const Date& o) const { return toDays() < o.toDays(); }
    bool operator<=(const Date& o) const { return toDays() <= o.toDays(); }
    Date operator+(int n) const { return fromDays(toDays() + n); }
    long operator-(const Date& o) const { return toDays() - o.toDays(); }
};
 
/* ============ CLASS: Transaction (abstract base) =========== */
class Transaction {
protected:
    int id;
    Date date;
    double amount;
    string description;
public:
    Transaction(int i, Date d, double a, string desc)
        : id(i), date(d), amount(a), description(desc) {}
    virtual ~Transaction() {}
 
    virtual string getType() const = 0;   /* "EXPENSE" or "INCOME" */
    virtual string getLabel() const = 0;  /* category or source   */
 
    virtual string serialize() const {
        ostringstream os;
        os << getType() << "|" << id << "|" << date.getDay() << "|" << date.getMonth()
           << "|" << date.getYear() << "|" << amount << "|" << getLabel() << "|" << description;
        return os.str();
    }
    virtual void display() const {
        cout << left << setw(5) << id << setw(12) << date.toString()
             << setw(9) << getType() << setw(14) << getLabel()
             << right << setw(10) << fixed << setprecision(2) << amount
             << "  " << left << description << endl;
    }
    int getId() const { return id; }
    Date getDate() const { return date; }
    double getAmount() const { return amount; }
    string getDescription() const { return description; }
};
 
/* ================= CLASS: Expense ========================== */
class Expense : public Transaction {
private:
    string category;
public:
    Expense(int i, Date d, double a, string cat, string desc)
        : Transaction(i, d, a, desc), category(cat) {}
    string getType() const { return "EXPENSE"; }
    string getLabel() const { return category; }
};
 
/* ================= CLASS: Income =========================== */
class Income : public Transaction {
private:
    string source;
public:
    Income(int i, Date d, double a, string src, string desc)
        : Transaction(i, d, a, desc), source(src) {}
    string getType() const { return "INCOME"; }
    string getLabel() const { return source; }
};
 
/* ============ Small helper functions for input ============= */
string readLine(const string& prompt) {
    string s;
    while (true) {
        cout << prompt;
        getline(cin, s);
        if (!s.empty()) {
            for (size_t i = 0; i < s.size(); i++) if (s[i] == '|') s[i] = '/';
            return s;
        }
        cout << "  Input cannot be empty.\n";
    }
}
 
int readInt(const string& prompt, int lo, int hi) {
    int v;
    while (true) {
        cout << prompt;
        if (cin >> v && v >= lo && v <= hi) { cin.ignore(10000, '\n'); return v; }
        cin.clear(); cin.ignore(10000, '\n');
        cout << "  Enter a whole number between " << lo << " and " << hi << ".\n";
    }
}
 
double readDouble(const string& prompt, double minv) {
    double v;
    while (true) {
        cout << prompt;
        if (cin >> v && v >= minv) { cin.ignore(10000, '\n'); return v; }
        cin.clear(); cin.ignore(10000, '\n');
        cout << "  Enter a valid amount (minimum " << minv << ").\n";
    }
}
 
Date readDate(const string& prompt) {
    while (true) {
        cout << prompt << " (DD MM YYYY, or 0 for today): ";
        int d;
        if (!(cin >> d)) { cin.clear(); cin.ignore(10000, '\n'); cout << "  Invalid.\n"; continue; }
        if (d == 0) { cin.ignore(10000, '\n'); return Date::today(); }
        int m, y;
        if (!(cin >> m >> y)) { cin.clear(); cin.ignore(10000, '\n'); cout << "  Invalid.\n"; continue; }
        cin.ignore(10000, '\n');
        Date dt(d, m, y);
        if (dt.isValid()) return dt;
        cout << "  That date does not exist. Try again.\n";
    }
}
 
vector<string> split(const string& s, char sep) {
    vector<string> parts; string cur;
    for (size_t i = 0; i < s.size(); i++) {
        if (s[i] == sep) { parts.push_back(cur); cur.clear(); }
        else cur += s[i];
    }
    parts.push_back(cur);
    return parts;
}
 
/* ============== CLASS: Budget ============================== */
class Budget {
private:
    double monthlyLimit;
public:
    Budget() : monthlyLimit(0) {}
    void setLimit(double l) { monthlyLimit = l; }
    double getLimit() const { return monthlyLimit; }
    void save() const { ofstream f(BUDGET_FILE.c_str()); f << monthlyLimit << endl; }
    void load() { ifstream f(BUDGET_FILE.c_str()); if (f) f >> monthlyLimit; }
 
    void check(double spent) const {
        if (monthlyLimit <= 0) { cout << "  No budget set yet (use menu option 7).\n"; return; }
        double pct = spent * 100.0 / monthlyLimit;
        cout << "  Budget: " << fixed << setprecision(2) << monthlyLimit
             << "  Spent: " << spent << "  (" << setprecision(1) << pct << "%)\n";
        if (spent > monthlyLimit)
            cout << "  *** ALERT: You have EXCEEDED your budget by "
                 << setprecision(2) << spent - monthlyLimit << " ***\n";
        else if (pct >= 80)
            cout << "  WARNING: More than 80% of the budget is used.\n";
        else
            cout << "  Good: You are within budget. Remaining "
                 << setprecision(2) << monthlyLimit - spent << "\n";
    }
};
 
/* ============== CLASS: ExpenseTracker (manager) ============ */
class ExpenseTracker {
private:
    vector<Transaction*> list;
    int nextId;
    ExpenseTracker(const ExpenseTracker&);             /* no copying */
    ExpenseTracker& operator=(const ExpenseTracker&);
public:
    ExpenseTracker() : nextId(1) {}
    ~ExpenseTracker() { for (size_t i = 0; i < list.size(); i++) delete list[i]; }
 
    void addExpense(Date d, double amt, const string& cat, const string& desc) {
        list.push_back(new Expense(nextId++, d, amt, cat, desc));
    }
    void addIncome(Date d, double amt, const string& src, const string& desc) {
        list.push_back(new Income(nextId++, d, amt, src, desc));
    }
 
    void showAll() const {
        if (list.empty()) { cout << "  No transactions yet.\n"; return; }
        cout << left << setw(5) << "ID" << setw(12) << "Date" << setw(9) << "Type"
             << setw(14) << "Category" << right << setw(10) << "Amount" << "  Description\n";
        cout << string(70, '-') << endl;
        for (size_t i = 0; i < list.size(); i++) list[i]->display();  /* polymorphism */
    }
 
    bool remove(int id) {
        for (size_t i = 0; i < list.size(); i++)
            if (list[i]->getId() == id) { delete list[i]; list.erase(list.begin() + i); return true; }
        return false;
    }
 
    void search(const string& key) const {
        string k = key; transform(k.begin(), k.end(), k.begin(), ::tolower);
        int found = 0;
        for (size_t i = 0; i < list.size(); i++) {
            string a = list[i]->getLabel() + " " + list[i]->getDescription();
            transform(a.begin(), a.end(), a.begin(), ::tolower);
            if (a.find(k) != string::npos) { list[i]->display(); found++; }
        }
        if (!found) cout << "  No matching records.\n";
    }
 
    double totalExpense(int month, int year) const {
        double t = 0;
        for (size_t i = 0; i < list.size(); i++)
            if (list[i]->getType() == "EXPENSE" && list[i]->getDate().getMonth() == month &&
                list[i]->getDate().getYear() == year) t += list[i]->getAmount();
        return t;
    }
 
    void monthlySummary(int month, int year) const {
        map<string, double> cat; double inc = 0, exp = 0;
        for (size_t i = 0; i < list.size(); i++) {
            Date d = list[i]->getDate();
            if (d.getMonth() != month || d.getYear() != year) continue;
            if (list[i]->getType() == "EXPENSE") { cat[list[i]->getLabel()] += list[i]->getAmount(); exp += list[i]->getAmount(); }
            else inc += list[i]->getAmount();
        }
        cout << "\n  ---- Summary for " << month << "/" << year << " ----\n";
        for (map<string, double>::iterator it = cat.begin(); it != cat.end(); ++it) {
            int bars = exp > 0 ? (int)(it->second * 30 / exp) : 0;
            cout << "  " << left << setw(14) << it->first << right << setw(10) << fixed << setprecision(2)
                 << it->second << "  " << string(bars, '#') << endl;
        }
        cout << "  ------------------------------------\n";
        cout << "  Total Income  : " << inc << "\n  Total Expense : " << exp
             << "\n  Net Balance   : " << inc - exp << endl;
    }
 
    /* total expense on one particular day (used by the unique feature) */
    double expenseOn(const Date& d) const {
        double t = 0;
        for (size_t i = 0; i < list.size(); i++)
            if (list[i]->getType() == "EXPENSE" && list[i]->getDate() == d) t += list[i]->getAmount();
        return t;
    }
    bool hasExpenseOn(const Date& d) const {
        for (size_t i = 0; i < list.size(); i++)
            if (list[i]->getType() == "EXPENSE" && list[i]->getDate() == d) return true;
        return false;
    }
    double averageSpendingDayAmount(const Date& from, const Date& to) const {
        map<long, double> perDay;
        for (size_t i = 0; i < list.size(); i++)
            if (list[i]->getType() == "EXPENSE" && from <= list[i]->getDate() && list[i]->getDate() <= to)
                perDay[list[i]->getDate().toDays()] += list[i]->getAmount();
        if (perDay.empty()) return 0;
        double sum = 0;
        for (map<long, double>::iterator it = perDay.begin(); it != perDay.end(); ++it) sum += it->second;
        return sum / perDay.size();
    }
 
    void save() const {
        ofstream f(DATA_FILE.c_str());
        for (size_t i = 0; i < list.size(); i++) f << list[i]->serialize() << "\n";
    }
    void load() {
        ifstream f(DATA_FILE.c_str());
        string line;
        while (getline(f, line)) {
            vector<string> p = split(line, '|');
            if (p.size() < 8) continue;
            int id = atoi(p[1].c_str());
            Date d(atoi(p[2].c_str()), atoi(p[3].c_str()), atoi(p[4].c_str()));
            double amt = atof(p[5].c_str());
            if (p[0] == "EXPENSE") list.push_back(new Expense(id, d, amt, p[6], p[7]));
            else list.push_back(new Income(id, d, amt, p[6], p[7]));
            if (id >= nextId) nextId = id + 1;
        }
    }
};
 
/* ==============================================================
 *  CLASS: StreakChallenge   <<< MY UNIQUE FEATURE >>>
 *  "No-Spend Streak Challenge": rewards the user for days on
 *  which no expense was recorded.
 * ============================================================== */
class StreakChallenge {
private:
    Date startDate;
    int targetDays;
    bool active;
 
    static string badgeFor(int best) {
        if (best >= 30) return "PLATINUM (30+ day streak)";
        if (best >= 14) return "GOLD (14+ day streak)";
        if (best >= 7)  return "SILVER (7+ day streak)";
        if (best >= 3)  return "BRONZE (3+ day streak)";
        return "No badge yet - keep going!";
    }
public:
    StreakChallenge() : startDate(Date::today()), targetDays(7), active(false) {}
    bool isActive() const { return active; }
 
    void setup() {
        startDate = readDate("  Challenge start date");
        targetDays = readInt("  Target streak in days (1-365): ", 1, 365);
        active = true;
        save();
        cout << "  Challenge started on " << startDate.toString() << "! Goal: " << targetDays
             << " no-spend days in a row.\n";
    }
    void stop() { active = false; save(); cout << "  Challenge stopped.\n"; }
 
    void save() const {
        ofstream f(CHALLENGE_FILE.c_str());
        f << active << " " << startDate.getDay() << " " << startDate.getMonth() << " "
          << startDate.getYear() << " " << targetDays << endl;
    }
    void load() {
        ifstream f(CHALLENGE_FILE.c_str());
        int a, d, m, y, t;
        if (f >> a >> d >> m >> y >> t) {
            Date dt(d, m, y);
            if (dt.isValid()) { active = (a == 1); startDate = dt; targetDays = t; }
        }
    }
 
    /* Main analysis report */
    void report(const ExpenseTracker& tr) const {
        if (!active) { cout << "  No active challenge. Choose 'Start new challenge' first.\n"; return; }
        Date today = Date::today();
        if (today < startDate) { cout << "  Challenge starts in the future (" << startDate.toString() << ").\n"; return; }
 
        long total = today - startDate + 1;
        int noSpend = 0, run = 0, best = 0, current = 0;
        Date brokenOn = startDate; bool everBroken = false;
        for (long i = 0; i < total; i++) {
            Date d = startDate + (int)i;
            if (!tr.hasExpenseOn(d)) { noSpend++; run++; if (run > best) best = run; }
            else { run = 0; brokenOn = d; everBroken = true; }
        }
        current = run;  /* streak ending today */
        double avg = tr.averageSpendingDayAmount(startDate, today);
        double saved = avg * noSpend;
        double pct = current * 100.0 / targetDays; if (pct > 100) pct = 100;
 
        cout << "\n  ======== NO-SPEND STREAK CHALLENGE ========\n";
        cout << "  Started         : " << startDate.toString() << "  (" << total << " days so far)\n";
        cout << "  Target          : " << targetDays << " days in a row\n";
        cout << "  No-spend days   : " << noSpend << " / " << total << "\n";
        cout << "  Current streak  : " << current << " day(s)\n";
        cout << "  Best streak     : " << best << " day(s)\n";
        if (everBroken) cout << "  Last spend day  : " << brokenOn.toString() << "\n";
        cout << "  Est. money saved: " << fixed << setprecision(2) << saved
             << "  (no-spend days x avg daily spend " << avg << ")\n";
        int filled = (int)(pct / 5);
        cout << "  Progress        : [" << string(filled, '=') << string(20 - filled, ' ') << "] "
             << setprecision(0) << pct << "%\n";
        cout << "  Badge earned    : " << badgeFor(best) << "\n";
        if (current >= targetDays) cout << "  *** CONGRATULATIONS! Challenge goal achieved! ***\n";
        else cout << "  " << targetDays - current << " more no-spend day(s) to reach the goal.\n";
    }
 
    /* Monthly calendar: * = no-spend day, # = spent money, . = outside challenge */
    void calendar(const ExpenseTracker& tr, int month, int year) const {
        if (!active) { cout << "  No active challenge.\n"; return; }
        static const char* names[] = {"January","February","March","April","May","June","July",
                                      "August","September","October","November","December"};
        cout << "\n  " << names[month - 1] << " " << year << "\n";
        cout << "   Mo   Tu   We   Th   Fr   Sa   Su\n" << right;
        Date first(1, month, year);
        int wd = (int)(((first.toDays() + 3) % 7 + 7) % 7);  /* 0 = Monday (1970-01-01 was Thursday) */
        cout << string(wd * 5, ' ');
        Date today = Date::today();
        for (int d = 1; d <= Date::daysInMonth(month, year); d++) {
            Date cur(d, month, year);
            char mark = '.';
            if (startDate <= cur && cur <= today) mark = tr.hasExpenseOn(cur) ? '#' : '*';
            cout << setw(3) << d << mark << " ";
            if ((wd + d) % 7 == 0) cout << "\n";
        }
        cout << "\n  Legend: * = no-spend day   # = spent money   . = not in challenge\n";
    }
};
 
/* ======================= MENUS ============================== */
void streakMenu(StreakChallenge& sc, const ExpenseTracker& tr) {
    while (true) {
        cout << "\n  --- NO-SPEND STREAK CHALLENGE ---\n"
             << "  1. Start new challenge\n  2. View my streak report\n"
             << "  3. View monthly calendar\n  4. Stop challenge\n  0. Back\n";
        int c = readInt("  Choice: ", 0, 4);
        if (c == 0) return;
        if (c == 1) sc.setup();
        else if (c == 2) sc.report(tr);
        else if (c == 3) {
            int m = readInt("  Month (1-12): ", 1, 12);
            int y = readInt("  Year: ", 1970, 2100);
            sc.calendar(tr, m, y);
        } else sc.stop();
    }
}
 
int main() {
    ExpenseTracker tracker;
    Budget budget;
    StreakChallenge challenge;
    tracker.load(); budget.load(); challenge.load();
 
    cout << "=========================================\n"
         << "      PERSONAL EXPENSE TRACKER\n"
         << "  Chandigarh University | BCA Data Science\n"
         << "=========================================\n";
 
    while (true) {
        cout << "\n  1. Add Expense\n  2. Add Income\n  3. View All Transactions\n"
             << "  4. Delete Transaction\n  5. Search\n  6. Monthly Summary\n"
             << "  7. Set / Check Monthly Budget\n  8. No-Spend Streak Challenge [NEW]\n"
             << "  0. Save & Exit\n";
        int ch = readInt("  Enter choice: ", 0, 8);
        if (ch == 0) break;
        switch (ch) {
            case 1: {
                Date d = readDate("  Date");
                double a = readDouble("  Amount: ", 0.01);
                string cat = readLine("  Category (Food/Travel/Rent/Shopping...): ");
                string desc = readLine("  Description: ");
                tracker.addExpense(d, a, cat, desc);
                cout << "  Expense added.\n";
                budget.check(tracker.totalExpense(d.getMonth(), d.getYear()));
                break; }
            case 2: {
                Date d = readDate("  Date");
                double a = readDouble("  Amount: ", 0.01);
                string src = readLine("  Source (Salary/Pocket money/Gift...): ");
                string desc = readLine("  Description: ");
                tracker.addIncome(d, a, src, desc);
                cout << "  Income added.\n";
                break; }
            case 3: tracker.showAll(); break;
            case 4: {
                int id = readInt("  Enter ID to delete: ", 1, 1000000);
                cout << (tracker.remove(id) ? "  Deleted.\n" : "  ID not found.\n");
                break; }
            case 5: tracker.search(readLine("  Keyword: ")); break;
            case 6: {
                int m = readInt("  Month (1-12): ", 1, 12);
                int y = readInt("  Year: ", 1970, 2100);
                tracker.monthlySummary(m, y);
                budget.check(tracker.totalExpense(m, y));
                break; }
            case 7: {
                cout << "  Current limit: " << budget.getLimit() << "\n";
                if (readInt("  1 = change limit, 2 = just check this month: ", 1, 2) == 1)
                    budget.setLimit(readDouble("  New monthly limit: ", 1));
                Date t = Date::today();
                budget.check(tracker.totalExpense(t.getMonth(), t.getYear()));
                break; }
            case 8: streakMenu(challenge, tracker); break;
        }
    }
    tracker.save(); budget.save(); challenge.save();
    cout << "  Data saved. Goodbye!\n";
    return 0;
}
  Personal-Expense-Tracker
