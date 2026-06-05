# Drone-Management-System
Object-Oriented Programming (OOP) principles based Drone Fleet Management System written in C++. Features flight simulation, battery management, and data persistence.

#include <iostream>
#include <vector>
#include <string>
#include <fstream>
#include <exception>
#include <memory>
#include <iomanip>
#include <sstream>
#include <limits>
#include <cmath>
#include <chrono>
#include <ctime>
#include <algorithm>

// ==========================================
// KONSOL RENK KODLARI (ANSI)
// ==========================================
#define RESET   "\033[0m"
#define RED     "\033[31m"
#define GREEN   "\033[32m"
#define YELLOW  "\033[33m"
#define BLUE    "\033[34m"
#define MAGENTA "\033[35m"
#define CYAN    "\033[36m"
#define WHITE   "\033[37m"
#define BOLD    "\033[1m"

// ==========================================
// DRONE DURUM ENUM
// ==========================================
enum class DroneStatus {
    Idle,       // Boşta
    OnMission,  // Görevde
    LowBattery  // Yetersiz Batarya
};

std::string statusToString(DroneStatus status) {
    switch (status) {
        case DroneStatus::Idle: return "Bosta";
        case DroneStatus::OnMission: return "Gorevde";
        case DroneStatus::LowBattery: return "Yetersiz Batarya";
        default: return "Bilinmiyor";
    }
}

// ==========================================
// SİSTEM LOGLAMA YAPISI (LOGGER SINGLETON)
// ==========================================
class Logger {
private:
    std::ofstream logFile;
    
    Logger() {
        // Online derleyicilerde dosya oluşturma hatasını engellemek için kontrol ekledik
        logFile.open("system.log", std::ios::app);
    }

    // Antigravity ve Cross-Platform uyumlu güvenli zaman damgası fonksiyonu
    std::string getTimestamp() const {
        auto now = std::chrono::system_clock::now();
        std::time_t now_time = std::chrono::system_clock::to_time_t(now);
        
        struct tm time_info;
#if defined(_MSC_VER)
        localtime_s(&time_info, &now_time);
#else
        // Linux ve GCC (Antigravity) uyumlu safe fonksiyon
        localtime_r(&now_time, &time_info);
#endif
        char buffer[20];
        std::strftime(buffer, sizeof(buffer), "%Y-%m-%d %H:%M:%S", &time_info);
        return std::string(buffer);
    }

public:
    static Logger& getInstance() {
        static Logger instance;
        return instance;
    }

    Logger(const Logger&) = delete;
    Logger& operator=(const Logger&) = delete;

    ~Logger() {
        if (logFile.is_open()) {
            logFile.close();
        }
    }

    void log(const std::string& level, const std::string& message) {
        std::string ts = getTimestamp();
        std::string formattedConsole = "";
        std::string formattedFile = "[" + ts + "] [" + level + "] " + message;

        if (level == "INFO") {
            formattedConsole = GREEN + std::string("[INFO] ") + RESET + message;
        } else if (level == "WARNING") {
            formattedConsole = YELLOW + std::string("[WARNING] ") + RESET + message;
        } else if (level == "ERROR") {
            formattedConsole = RED + std::string("[ERROR] ") + RESET + message;
        } else {
            formattedConsole = "[" + level + "] " + message;
        }

        std::cout << formattedConsole << std::endl;

        if (logFile.is_open()) {
            logFile << formattedFile << std::endl;
            logFile.flush();
        }
    }

    void info(const std::string& message) { log("INFO", message); }
    void warn(const std::string& message) { log("WARNING", message); }
    void error(const std::string& message) { log("ERROR", message); }
};

// ==========================================
// HATA KONTROLÜ (EXCEPTION HANDLING)
// ==========================================
class DroneException : public std::exception {
protected:
    std::string message;
public:
    explicit DroneException(std::string msg) : message(std::move(msg)) {}
    const char* what() const noexcept override {
        return message.c_str();
    }
};

class BatteryLowException : public DroneException {
public:
    explicit BatteryLowException(const std::string& msg) : DroneException(msg) {}
};

class InvalidDroneIdException : public DroneException {
public:
    explicit InvalidDroneIdException(const std::string& msg) : DroneException(msg) {}
};

class FileException : public DroneException {
public:
    explicit FileException(const std::string& msg) : DroneException(msg) {}
};

class InvalidInputException : public DroneException {
public:
    explicit InvalidInputException(const std::string& msg) : DroneException(msg) {}
};

// ==========================================
// INTERFACE (TASK EXECUTOR)
// ==========================================
class TaskExecutor {
public:
    virtual ~TaskExecutor() = default;
    virtual void executeTask(const std::string& taskName) = 0;
};

// ==========================================
// ENCAPSULATION & BASE CLASS (DRONE)
// ==========================================
class Drone : public TaskExecutor {
protected:
    int id;
    std::string model;
    double speed;          
    double altitude;       
    double x, y;          
    double battery;        
    DroneStatus status;
    std::string currentTask;

    double targetX, targetY;
    double targetSpeed, targetAltitude;

public:
    Drone(int droneId, std::string droneModel)
        : id(droneId), model(std::move(droneModel)), speed(0.0), altitude(0.0),
          x(0.0), y(0.0), battery(100.0), status(DroneStatus::Idle), currentTask("Yok"),
          targetX(0.0), targetY(0.0), targetSpeed(0.0), targetAltitude(0.0) {}

    virtual ~Drone() = default;

    virtual void uCusYap(double tx, double ty, double s, double alt) = 0;
    virtual void telemetriRaporla() const = 0;
    virtual std::string getDroneType() const = 0;

    void executeTask(const std::string& taskName) override {
        if (battery < 20.0) {
            status = DroneStatus::LowBattery;
            throw BatteryLowException("Drone ID " + std::to_string(id) + " yetersiz batarya seviyesine sahip (%" + 
                                      std::to_string(static_cast<int>(battery)) + "). Gorev atanamaz!");
        }
        currentTask = taskName;
        status = DroneStatus::OnMission;
        Logger::getInstance().info("Drone ID " + std::to_string(id) + " '" + taskName + "' gorevine basladi.");
    }

    void charge() {
        battery = 100.0;
        if (status == DroneStatus::LowBattery) {
            status = DroneStatus::Idle;
        }
        Logger::getInstance().info("Drone ID " + std::to_string(id) + " bataryasi tamamen sarj edildi (%100).");
    }

    void updateBattery(double drainAmount) {
        battery -= drainAmount;
        if (battery < 0.0) battery = 0.0;
        
        if (battery < 20.0 && status != DroneStatus::LowBattery) {
            status = DroneStatus::LowBattery;
            Logger::getInstance().warn("Drone ID " + std::to_string(id) + " bataryasi kritik seviyenin altina dustu! (%20 altinda)");
        }
    }

    // Getters & Setters
    int getId() const { return id; }
    std::string getModel() const { return model; }
    double getSpeed() const { return speed; }
    double getAltitude() const { return altitude; }
    double getX() const { return x; }
    double getY() const { return y; }
    double getBattery() const { return battery; }
    DroneStatus getStatus() const { return status; }
    std::string getStatusString() const { return statusToString(status); }
    std::string getCurrentTask() const { return currentTask; }
    double getTargetX() const { return targetX; }
    double getTargetY() const { return targetY; }
    double getTargetSpeed() const { return targetSpeed; }
    double getTargetAltitude() const { return targetAltitude; }

    void setSpeed(double s) { speed = s; }
    void setAltitude(double alt) { altitude = alt; }
    void setCoordinates(double cx, double cy) { x = cx; y = cy; }
    void setStatus(DroneStatus s) { status = s; }
    void setCurrentTask(const std::string& task) { currentTask = task; }
    void setTargets(double tx, double ty, double ts, double ta) {
        targetX = tx; targetY = ty; targetSpeed = ts; targetAltitude = ta;
    }
    void setBattery(double bat) {
        battery = std::max(0.0, std::min(100.0, bat));
        if (battery < 20.0) { status = DroneStatus::LowBattery; }
    }
};

// ==========================================
// DERIVED CLASS (KEŞİF DRONU)
// ==========================================
class ReconnaissanceDrone : public Drone {
private:
    std::string cameraResolution;
    bool hasNightVision;

public:
    ReconnaissanceDrone(int droneId, std::string droneModel, std::string resolution, bool nightVision)
        : Drone(droneId, std::move(droneModel)), cameraResolution(std::move(resolution)), hasNightVision(nightVision) {}

    void uCusYap(double tx, double ty, double s, double alt) override {
        setTargets(tx, ty, s, alt);
        double dx = tx - x;
        double dy = ty - y;
        double distance = std::sqrt(dx*dx + dy*dy);

        if (distance < 0.1) {
            speed = 0.0; altitude = 0.0;
            status = DroneStatus::Idle; currentTask = "Yok";
            Logger::getInstance().info("Kesif Dronu (ID: " + std::to_string(id) + ") gorev koordinatina ulasti.");
            return;
        }

        double step = s / 5.0; 
        if (step > distance) step = distance;

        double ratio = step / distance;
        x += dx * ratio;
        y += dy * ratio;
        speed = s;
        altitude = alt;

        double baseDrain = 3.0 + (alt * 0.05) + (s * 0.1);
        if (hasNightVision) { baseDrain += 1.5; }
        
        updateBattery(baseDrain);

        Logger::getInstance().info("Kesif Dronu (ID: " + std::to_string(id) + ") ucuyor. Konum: (" +
                                   std::to_string(x) + ", " + std::to_string(y) + ") | Batarya: %" + std::to_string(static_cast<int>(battery)));
    }

    void telemetriRaporla() const override {
        std::cout << CYAN << BOLD << "--- KESIS DRONU TELEMETRI VE BILGI RAPORU ---" << RESET << std::endl;
        std::cout << "ID             : " << id << "\nModel          : " << model << "\nTur            : Kesif Dronu\n";
        std::cout << "Koordinat      : (" << std::fixed << std::setprecision(2) << x << ", " << y << ")\n";
        std::cout << "Hiz / Irtifa   : " << speed << " m/s / " << altitude << " m\n";
        std::cout << "Batarya        : %" << static_cast<int>(battery) << (battery < 20.0 ? RED " [KRITIK]" RESET : "") << "\n";
        std::cout << "Durum          : " << getStatusString() << "\nMevcut Gorev   : " << currentTask << "\n";
        std::cout << "Kamera Cozun.  : " << cameraResolution << "\nGece Gorusu    : " << (hasNightVision ? "Mevcut" : "Mevcut Degil") << "\n";
        std::cout << "--------------------------------------------" << RESET << std::endl;
    }

    std::string getDroneType() const override { return "RECON"; }
    std::string getCameraResolution() const { return cameraResolution; }
    bool getHasNightVision() const { return hasNightVision; }
};

// ==========================================
// DERIVED CLASS (KARGO DRONU)
// ==========================================
class CargoDrone : public Drone {
private:
    double cargoCapacity;        
    double currentCargoWeight;   

public:
    CargoDrone(int droneId, std::string droneModel, double capacity, double weight = 0.0)
        : Drone(droneId, std::move(droneModel)), cargoCapacity(capacity), currentCargoWeight(weight) {}

    void loadCargo(double weight) {
        if (weight < 0.0) { throw InvalidInputException("Yuk miktari negatif olamaz!"); }
        if (currentCargoWeight + weight > cargoCapacity) {
            throw InvalidInputException("Kargo kapasitesi asildi! Maks: " + std::to_string(cargoCapacity) + " kg");
        }
        currentCargoWeight += weight;
        Logger::getInstance().info("Kargo Dronu (ID: " + std::to_string(id) + ") " + std::to_string(weight) + " kg yuklendi.");
    }

    void unloadCargo() {
        if (currentCargoWeight > 0.0) {
            Logger::getInstance().info("Kargo Dronu (ID: " + std::to_string(id) + ") " + std::to_string(currentCargoWeight) + " kg bosaltildi.");
            currentCargoWeight = 0.0;
        }
    }

    void uCusYap(double tx, double ty, double s, double alt) override {
        setTargets(tx, ty, s, alt);
        double dx = tx - x;
        double dy = ty - y;
        double distance = std::sqrt(dx*dx + dy*dy);

        if (distance < 0.1) {
            speed = 0.0; altitude = 0.0; status = DroneStatus::Idle; currentTask = "Yok";
            Logger::getInstance().info("Kargo Dronu (ID: " + std::to_string(id) + ") teslimati tamamladi.");
            unloadCargo(); 
            return;
        }

        double step = s / 5.0; 
        if (step > distance) step = distance;

        double ratio = step / distance;
        x += dx * ratio;
        y += dy * ratio;
        speed = s;
        altitude = alt;

        double weightFactor = 1.0 + (currentCargoWeight / cargoCapacity);
        double baseDrain = (5.0 + (alt * 0.08) + (s * 0.15)) * weightFactor;

        updateBattery(baseDrain);

        Logger::getInstance().info("Kargo Dronu (ID: " + std::to_string(id) + ") ucuyor. Konum: (" +
                                   std::to_string(x) + ", " + std::to_string(y) + ") | Batarya: %" + std::to_string(static_cast<int>(battery)));
    }

    void telemetriRaporla() const override {
        std::cout << MAGENTA << BOLD << "--- KARGO DRONU TELEMETRI VE BILGI RAPORU ---" << RESET << std::endl;
        std::cout << "ID             : " << id << "\nModel          : " << model << "\nTur            : Kargo Dronu\n";
        std::cout << "Koordinat      : (" << std::fixed << std::setprecision(2) << x << ", " << y << ")\n";
        std::cout << "Hiz / Irtifa   : " << speed << " m/s / " << altitude << " m\n";
        std::cout << "Batarya        : %" << static_cast<int>(battery) << (battery < 20.0 ? RED " [KRITIK]" RESET : "") << "\n";
        std::cout << "Durum          : " << getStatusString() << "\nMevcut Gorev   : " << currentTask << "\n";
        std::cout << "Kapasite       : " << cargoCapacity << " kg\nMevcut Yuk     : " << currentCargoWeight << " kg\n";
        std::cout << "--------------------------------------------" << RESET << std::endl;
    }

    std::string getDroneType() const override { return "CARGO"; }
    double getCargoCapacity() const { return cargoCapacity; }
    double getCurrentCargoWeight() const { return currentCargoWeight; }
};

// ==========================================
// FILO YÖNETİCİSİ (FLEET MANAGER)
// ==========================================
class FleetManager {
private:
    std::vector<Drone*> drones;

public:
    FleetManager() = default;

    ~FleetManager() {
        for (Drone* drone : drones) { delete drone; }
        drones.clear();
    }

    void addDrone(Drone* drone) {
        if (drone == nullptr) return;
        for (const auto& d : drones) {
            if (d->getId() == drone->getId()) {
                delete drone; 
                throw InvalidInputException("Hata: " + std::to_string(drone->getId()) + " ID zaten kayitli!");
            }
        }
        drones.push_back(drone);
        Logger::getInstance().info("Yeni Drone eklendi. ID: " + std::to_string(drone->getId()));
    }

    Drone* findDrone(int id) {
        for (auto& drone : drones) {
            if (drone->getId() == id) return drone;
        }
        throw InvalidDroneIdException("Hata: ID'si " + std::to_string(id) + " olan drone bulunamadi!");
    }

    void listFleet() {
        if (drones.empty()) {
            std::cout << YELLOW << "Filoda kayitli drone bulunmamaktadir." << RESET << std::endl;
            return;
        }
        std::cout << BOLD << "\n================= DRONE FILOSU LISTESI =================\n" << RESET;
        for (size_t i = 0; i < drones.size(); ++i) {
            std::cout << "[" << i + 1 << "] ";
            drones[i]->telemetriRaporla();
            std::cout << std::endl;
        }
    }

    void assignTask(int id, const std::string& taskName, double tx, double ty, double ts, double ta) {
        Drone* drone = findDrone(id); 
        drone->executeTask(taskName); 
        drone->setTargets(tx, ty, ts, ta);
    }

    void simulateStep() {
        bool activeMission = false;
        Logger::getInstance().info("Simulasyon adimi ilerletiliyor...");

        for (auto& drone : drones) {
            if (drone->getStatus() == DroneStatus::OnMission) {
                activeMission = true;
                if (drone->getBattery() <= 0.0) {
                    Logger::getInstance().error("Drone ID " + std::to_string(drone->getId()) + " kaza yapti (Batarya %0).");
                    drone->setStatus(DroneStatus::LowBattery);
                    drone->setSpeed(0.0); drone->setAltitude(0.0);
                    drone->setCurrentTask("Zorunlu Inis");
                    continue;
                }
                drone->uCusYap(drone->getTargetX(), drone->getTargetY(), drone->getTargetSpeed(), drone->getTargetAltitude());
            }
        }
        if (!activeMission) {
            Logger::getInstance().warn("Gorevde olan drone yok. Konumlar sabit.");
        }
    }

    void chargeAll() {
        for (auto& drone : drones) { drone->charge(); }
    }

    void saveReport(const std::string& filename) {
        std::ofstream reportFile(filename);
        if (!reportFile.is_open()) return; // Hata fırlatmak yerine online compiler'lar için esnek geçiş

        reportFile << "=======================================================\n";
        reportFile << "               DRONE FILO YONETIM RAPORU               \n";
        reportFile << "=======================================================\n";
        reportFile << "Toplam Drone Sayisi: " << drones.size() << "\n\n";

        for (const auto& drone : drones) {
            reportFile << "ID: " << drone->getId() << " | Model: " << drone->getModel() << " | Tip: " << drone->getDroneType() << "\n";
            reportFile << "  Durum: " << drone->getStatusString() << " | Batarya: %" << static_cast<int>(drone->getBattery()) << "\n";
            reportFile << "-------------------------------------------------------\n";
        }
        reportFile.close();
    }

    void saveFleetState(const std::string& filename) {
        std::ofstream file(filename);
        if (!file.is_open()) return;

        for (const auto& drone : drones) {
            if (drone->getDroneType() == "RECON") {
                auto* rd = dynamic_cast<ReconnaissanceDrone*>(drone);
                if (rd) {
                    file << "RECON," << rd->getId() << "," << rd->getModel() << "," << rd->getBattery() << ","
                         << rd->getX() << "," << rd->getY() << "," << rd->getSpeed() << "," << rd->getAltitude() << ","
                         << static_cast<int>(rd->getStatus()) << "," << rd->getCurrentTask() << ","
                         << rd->getTargetX() << "," << rd->getTargetY() << "," << rd->getTargetSpeed() << "," << rd->getTargetAltitude() << ","
                         << rd->getCameraResolution() << "," << (rd->getHasNightVision() ? "1" : "0") << "\n";
                }
            } else if (drone->getDroneType() == "CARGO") {
                auto* cd = dynamic_cast<CargoDrone*>(drone);
                if (cd) {
                    file << "CARGO," << cd->getId() << "," << cd->getModel() << "," << cd->getBattery() << ","
                         << cd->getX() << "," << cd->getY() << "," << cd->getSpeed() << "," << cd->getAltitude() << ","
                         << static_cast<int>(cd->getStatus()) << "," << cd->getCurrentTask() << ","
                         << cd->getTargetX() << "," << cd->getTargetY() << "," << cd->getTargetSpeed() << "," << cd->getTargetAltitude() << ","
                         << cd->getCargoCapacity() << "," << cd->getCurrentCargoWeight() << "\n";
                }
            }
        }
        file.close();
    }

    void loadFleetState(const std::string& filename) {
        std::ifstream file(filename);
        if (!file.is_open()) return;

        std::string line;
        while (std::getline(file, line)) {
            if (line.empty()) continue;
            std::stringstream ss(line);
            std::string type;
            std::getline(ss, type, ',');

            try {
                if (type == "RECON") {
                    std::string idStr, modelVal, batStr, xStr, yStr, speedStr, altStr, statStr, taskVal, txStr, tyStr, tsStr, taStr, resVal, nvStr;
                    std::getline(ss, idStr, ','); std::getline(ss, modelVal, ','); std::getline(ss, batStr, ',');
                    std::getline(ss, xStr, ','); std::getline(ss, yStr, ','); std::getline(ss, speedStr, ',');
                    std::getline(ss, altStr, ','); std::getline(ss, statStr, ','); std::getline(ss, taskVal, ',');
                    std::getline(ss, txStr, ','); std::getline(ss, tyStr, ','); std::getline(ss, tsStr, ',');
                    std::getline(ss, taStr, ','); std::getline(ss, resVal, ','); std::getline(ss, nvStr, ',');

                    auto* rd = new ReconnaissanceDrone(std::stoi(idStr), modelVal, resVal, (nvStr == "1"));
                    rd->setBattery(std::stod(batStr)); rd->setCoordinates(std::stod(xStr), std::stod(yStr));
                    rd->setSpeed(std::stod(speedStr)); rd->setAltitude(std::stod(altStr));
                    rd->setStatus(static_cast<DroneStatus>(std::stoi(statStr))); rd->setCurrentTask(taskVal);
                    rd->setTargets(std::stod(txStr), std::stod(tyStr), std::stod(tsStr), std::stod(taStr));
                    drones.push_back(rd);
                } else if (type == "CARGO") {
                    std::string idStr, modelVal, batStr, xStr, yStr, speedStr, altStr, statStr, taskVal, txStr, tyStr, tsStr, taStr, capStr, wStr;
                    std::getline(ss, idStr, ','); std::getline(ss, modelVal, ','); std::getline(ss, batStr, ',');
                    std::getline(ss, xStr, ','); std::getline(ss, yStr, ','); std::getline(ss, speedStr, ',');
                    std::getline(ss, altStr, ','); std::getline(ss, statStr, ','); std::getline(ss, taskVal, ',');
                    std::getline(ss, txStr, ','); std::getline(ss, tyStr, ','); std::getline(ss, tsStr, ',');
                    std::getline(ss, taStr, ','); std::getline(ss, capStr, ','); std::getline(ss, wStr, ',');

                    auto* cd = new CargoDrone(std::stoi(idStr), modelVal, std::stod(capStr), std::stod(wStr));
                    cd->setBattery(std::stod(batStr)); cd->setCoordinates(std::stod(xStr), std::stod(yStr));
                    cd->setSpeed(std::stod(speedStr)); cd->setAltitude(std::stod(altStr));
                    cd->setStatus(static_cast<DroneStatus>(std::stoi(statStr))); cd->setCurrentTask(taskVal);
                    cd->setTargets(std::stod(txStr), std::stod(tyStr), std::stod(tsStr), std::stod(taStr));
                    drones.push_back(cd);
                }
            } catch (...) {}
        }
        file.close();
    }

    const std::vector<Drone*>& getDrones() const { return drones; }
};

// ==========================================
// VERİ DOĞRULAMA (INPUT VALIDATION)
// ==========================================
void clearInputBuffer() {
    std::cin.clear();
    std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
}

int getValidInt(const std::string& prompt, int minVal = std::numeric_limits<int>::min(), int maxVal = std::numeric_limits<int>::max()) {
    int val;
    while (true) {
        std::cout << prompt;
        if (std::cin >> val) {
            if (val >= minVal && val <= maxVal) {
                clearInputBuffer();
                return val;
            }
        }
        std::cout << RED << "Gecersiz Girdi!" << RESET << std::endl;
        clearInputBuffer();
    }
}

double getValidDouble(const std::string& prompt, double minVal = 0.0, double maxVal = std::numeric_limits<double>::max()) {
    double val;
    while (true) {
        std::cout << prompt;
        if (std::cin >> val) {
            if (val >= minVal && val <= maxVal) {
                clearInputBuffer();
                return val;
            }
        }
        std::cout << RED << "Gecersiz Girdi!" << RESET << std::endl;
        clearInputBuffer();
    }
}

std::string getValidString(const std::string& prompt) {
    std::string val;
    while (true) {
        std::cout << prompt;
        std::getline(std::cin, val);
        if (!val.empty()) return val;
    }
}

bool getValidBool(const std::string& prompt) {
    while (true) {
        std::string val = getValidString(prompt + " (E/H): ");
        if (val == "E" || val == "e" || val == "Evet" || val == "evet") return true;
        if (val == "H" || val == "h" || val == "Hayir" || val == "hayir") return false;
    }
}

// ==========================================
// MAIN LOOP
// ==========================================
int main() {
    Logger::getInstance().info("Drone Filo Yonetim Sistemi Antigravity Modunda Baslatildi.");
    FleetManager fleet;

    // Dosya okuma sistemi sunucu kısıtlamalarına göre güvenli hale getirildi
    fleet.loadFleetState("filo_durumu.txt");

    bool running = true;
    while (running) {
        std::cout << "\n" << BOLD << BLUE << "=======================================================\n" << RESET;
        std::cout << "  1. Filodaki Dronlari Listele\n";
        std::cout << "  2. Yeni Drone Ekle\n";
        std::cout << "  3. Drona Gorev Ata\n";
        std::cout << "  4. Simulasyonu Baslat / Adim Ilerlet\n";
        std::cout << "  5. Tum Dronlari Sarj Et\n";
        std::cout << "  6. Rapor Kaydet ve Cikis Yap\n";
        std::cout << BOLD << BLUE << "=======================================================\n" << RESET;

        int choice = getValidInt("Seciminiz (1-6): ", 1, 6);

        switch (choice) {
            case 1: fleet.listFleet(); break;
            case 2: {
                std::cout << "\n1. Kesif Dronu | 2. Kargo Dronu\n";
                int typeChoice = getValidInt("Tip Secin: ", 1, 2);
                try {
                    int id = getValidInt("Unique ID: ", 1);
                    std::string model = getValidString("Model Name: ");
                    if (typeChoice == 1) {
                        std::string res = getValidString("Kamera Coz: ");
                        bool nv = getValidBool("Gece Gorusu Var mi?");
                        fleet.addDrone(new ReconnaissanceDrone(id, model, res, nv));
                    } else {
                        double cap = getValidDouble("Kapasite (kg): ", 0.1);
                        fleet.addDrone(new CargoDrone(id, model, cap, 0.0));
                    }
                } catch (const std::exception& e) { Logger::getInstance().error(e.what()); }
                break;
            }
            case 3: {
                if (fleet.getDrones().empty()) { std::cout << "Filo bos!\n"; break; }
                int id = getValidInt("Drone ID: ", 1);
                try {
                    Drone* drone = fleet.findDrone(id);
                    std::string task = getValidString("Gorev Adi: ");
                    double tx = getValidDouble("Hedef X: ", -1000.0, 1000.0);
                    double ty = getValidDouble("Hedef Y: ", -1000.0, 1000.0);
                    double ts = getValidDouble("Hiz (m/s): ", 1.0, 100.0);
                    double ta = getValidDouble("Irtifa (m): ", 1.0, 1000.0);

                    if (drone->getDroneType() == "CARGO") {
                        auto* cd = dynamic_cast<CargoDrone*>(drone);
                        if (cd && getValidBool("Yuk eklemek ister misiniz?")) {
                            double w = getValidDouble("Agirlik: ", 0.1, cd->getCargoCapacity());
                            cd->loadCargo(w);
                        }
                    }
                    fleet.assignTask(id, task, tx, ty, ts, ta);
                } catch (const std::exception& e) { Logger::getInstance().error(e.what()); }
                break;
            }
            case 4: {
                int steps = getValidInt("Adim Sayisi (1-10): ", 1, 10);
                for (int i = 0; i < steps; ++i) { fleet.simulateStep(); }
                break;
            }
            case 5: fleet.chargeAll(); break;
            case 6: {
                fleet.saveReport("rapor.txt");
                fleet.saveFleetState("filo_durumu.txt");
                Logger::getInstance().info("Sistem guvenli kapatildi.");
                running = false;
                break;
            }
        }
    }
    return 0;
}
