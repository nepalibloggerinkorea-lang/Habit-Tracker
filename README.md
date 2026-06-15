
import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';

void main() => runApp(const HabitTrackerApp());

class HabitTrackerApp extends StatefulWidget {
  const HabitTrackerApp({super.key});

  @override
  State<HabitTrackerApp> createState() => _HabitTrackerAppState();
}

class _HabitTrackerAppState extends State<HabitTrackerApp> {
  bool darkMode = false;

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      themeMode: darkMode ? ThemeMode.dark : ThemeMode.light,
      theme: ThemeData(useMaterial3: true, colorSchemeSeed: Colors.deepPurple),
      darkTheme: ThemeData.dark(useMaterial3: true),
      home: LoginPage(
        onToggleTheme: () => setState(() => darkMode = !darkMode),
      ),
    );
  }
}

class User {
  String username;
  String password;

  User(this.username, this.password);

  Map<String, dynamic> toJson() => {
        'username': username,
        'password': password,
      };

  factory User.fromJson(Map<String, dynamic> json) =>
      User(json['username'], json['password']);
}

class Habit {
  String name;
  String frequency;
  bool completed;

  Habit({
    required this.name,
    required this.frequency,
    this.completed = false,
  });

  Map<String, dynamic> toJson() => {
        'name': name,
        'frequency': frequency,
        'completed': completed,
      };

  factory Habit.fromJson(Map<String, dynamic> json) => Habit(
        name: json['name'],
        frequency: json['frequency'],
        completed: json['completed'],
      );
}

class LoginPage extends StatefulWidget {
  final VoidCallback onToggleTheme;
  const LoginPage({super.key, required this.onToggleTheme});

  @override
  State<LoginPage> createState() => _LoginPageState();
}

class _LoginPageState extends State<LoginPage> {
  final user = TextEditingController();
  final pass = TextEditingController();

  Future<void> login() async {
    final prefs = await SharedPreferences.getInstance();
    final data = prefs.getString('user');

    if (data == null) {
      show('Please register first');
      return;
    }

    final saved = User.fromJson(jsonDecode(data));

    if (saved.username == user.text && saved.password == pass.text) {
      Navigator.pushReplacement(
        context,
        MaterialPageRoute(
          builder: (_) => HabitPage(username: saved.username),
        ),
      );
    } else {
      show('Invalid login');
    }
  }

  void show(String msg) {
    ScaffoldMessenger.of(context)
        .showSnackBar(SnackBar(content: Text(msg)));
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Habit Tracker'),
        actions: [
          IconButton(
            icon: const Icon(Icons.dark_mode),
            onPressed: widget.onToggleTheme,
          )
        ],
      ),
      body: Padding(
        padding: const EdgeInsets.all(20),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Icon(Icons.track_changes, size: 100),
            const SizedBox(height: 20),
            TextField(
              controller: user,
              decoration: const InputDecoration(labelText: 'Username'),
            ),
            TextField(
              controller: pass,
              obscureText: true,
              decoration: const InputDecoration(labelText: 'Password'),
            ),
            const SizedBox(height: 20),
            FilledButton(onPressed: login, child: const Text("Login")),
            TextButton(
              child: const Text("Register"),
              onPressed: () => Navigator.push(
                context,
                MaterialPageRoute(builder: (_) => const RegisterPage()),
              ),
            )
          ],
        ),
      ),
    );
  }
}

class RegisterPage extends StatefulWidget {
  const RegisterPage({super.key});

  @override
  State<RegisterPage> createState() => _RegisterPageState();
}

class _RegisterPageState extends State<RegisterPage> {
  final user = TextEditingController();
  final pass = TextEditingController();

  Future<void> register() async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString(
      'user',
      jsonEncode(User(user.text, pass.text).toJson()),
    );

    if (mounted) Navigator.pop(context);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text("Register")),
      body: Padding(
        padding: const EdgeInsets.all(20),
        child: Column(
          children: [
            TextField(controller: user, decoration: const InputDecoration(labelText: 'Username')),
            TextField(controller: pass, obscureText: true, decoration: const InputDecoration(labelText: 'Password')),
            const SizedBox(height: 20),
            FilledButton(onPressed: register, child: const Text("Create Account"))
          ],
        ),
      ),
    );
  }
}

class HabitPage extends StatefulWidget {
  final String username;
  const HabitPage({super.key, required this.username});

  @override
  State<HabitPage> createState() => _HabitPageState();
}

class _HabitPageState extends State<HabitPage> {
  List<Habit> habits = [];

  @override
  void initState() {
    super.initState();
    loadHabits();
  }

  Future<void> loadHabits() async {
    final prefs = await SharedPreferences.getInstance();
    final saved = prefs.getStringList('habits') ?? [];

    setState(() {
      habits = saved.map((e) => Habit.fromJson(jsonDecode(e))).toList();
    });
  }

  Future<void> saveHabits() async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setStringList(
      'habits',
      habits.map((e) => jsonEncode(e.toJson())).toList(),
    );
  }

  void addHabit() {
    final controller = TextEditingController();
    String frequency = "Daily";

    showDialog(
      context: context,
      builder: (_) => StatefulBuilder(
        builder: (context, setDialog) => AlertDialog(
          title: const Text("New Habit"),
          content: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              TextField(controller: controller),
              const SizedBox(height: 10),
              DropdownButton<String>(
                value: frequency,
                isExpanded: true,
                items: const [
                  DropdownMenuItem(value: "Daily", child: Text("Daily")),
                  DropdownMenuItem(value: "Weekly", child: Text("Weekly")),
                  DropdownMenuItem(value: "Monthly", child: Text("Monthly")),
                ],
                onChanged: (v) => setDialog(() => frequency = v!),
              )
            ],
          ),
          actions: [
            FilledButton(
              onPressed: () {
                if (controller.text.isNotEmpty) {
                  setState(() {
                    habits.add(
                      Habit(name: controller.text, frequency: frequency),
                    );
                  });
                  saveHabits();
                  Navigator.pop(context);
                }
              },
              child: const Text("Add"),
            )
          ],
        ),
      ),
    );
  }

  void deleteHabit(int index) {
    setState(() => habits.removeAt(index));
    saveHabits();
  }

  @override
  Widget build(BuildContext context) {
    int completed = habits.where((h) => h.completed).length;

    return Scaffold(
      appBar: AppBar(
        title: Text("Welcome ${widget.username}"),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: addHabit,
        child: const Icon(Icons.add),
      ),
      body: Column(
        children: [
          Container(
            margin: const EdgeInsets.all(16),
            padding: const EdgeInsets.all(20),
            decoration: BoxDecoration(
              borderRadius: BorderRadius.circular(20),
              gradient: const LinearGradient(
                colors: [Colors.deepPurple, Colors.purple],
              ),
            ),
            child: Column(
              children: [
                const Text(
                  "Progress",
                  style: TextStyle(color: Colors.white, fontSize: 18),
                ),
                Text(
                  "$completed / ${habits.length}",
                  style: const TextStyle(
                    color: Colors.white,
                    fontSize: 32,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                const SizedBox(height: 10),
                LinearProgressIndicator(
                  value: habits.isEmpty ? 0 : completed / habits.length,
                )
              ],
            ),
          ),
          Expanded(
            child: ListView.builder(
              itemCount: habits.length,
              itemBuilder: (context, index) {
                final habit = habits[index];

                return Card(
                  margin: const EdgeInsets.symmetric(
                      horizontal: 12, vertical: 6),
                  child: ListTile(
                    leading: Checkbox(
                      value: habit.completed,
                      onChanged: (v) {
                        setState(() => habit.completed = v!);
                        saveHabits();
                      },
                    ),
                    title: Text(habit.name),
                    subtitle: Text(habit.frequency),
                    trailing: IconButton(
                      icon: const Icon(Icons.delete),
                      onPressed: () => deleteHabit(index),
                    ),
                  ),
                );
              },
            ),
          )
        ],
      ),
    );
  }
}
