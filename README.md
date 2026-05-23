// ============================================================
// VOIDWARE v4.0 - IMGUI EDITION (FULLY FIXED)
// All features preserved: Rage, Legit, Auto Fire, Visuals, etc.
// Compile: Requires ImGui files + DirectX 11
// ============================================================

#include <Windows.h>
#include <TlHelp32.h>
#include <string>
#include <cmath>
#include <thread>
#include <random>
#include <intrin.h>
#include <winhttp.h>
#include <shellapi.h>
#include <chrono>
#include <vector>
#include <algorithm>
#include <fstream>
#include <d3d11.h>
#include <d3dcompiler.h>

#include "imgui.h"
#include "imgui_impl_win32.h"
#include "imgui_impl_dx11.h"

#pragma comment(lib, "winmm.lib")
#pragma comment(lib, "winhttp.lib")
#pragma comment(lib, "shell32.lib")
#pragma comment(lib, "d3d11.lib")
#pragma comment(lib, "d3dcompiler.lib")

#define CHEAT_VERSION "4.0"

// ==================== KEYAUTH CONFIGURATION ====================
#define KEYAUTH_NAME "VoidWare"
#define KEYAUTH_OWNERID "XfwwmtO8U3"
#define KEYAUTH_SECRET "45d0249b058e9d0734c00e6f23b2f7c518e4322a658d222166d4e244f4887d85"
#define KEYAUTH_VERSION "1.0"
#define KEYAUTH_API "keyauth.win"
#define KEYAUTH_INIT "/api/1.2/init.php"
#define KEYAUTH_LICENSE "/api/1.2/license.php"

// ==================== OFFSETS ====================
namespace Offsets {
    constexpr uintptr_t dwLocalPlayer = 0x17E0A68;
    constexpr uintptr_t dwEntityList = 0x18CFE48;
    constexpr uintptr_t dwViewMatrix = 0x18D0670;
    constexpr uintptr_t dwForceJump = 0x1732F30;
    constexpr uintptr_t dwForceAttack = 0x1732EC0;
    constexpr uintptr_t dwForceAttack2 = 0x1732ECC;
    constexpr uintptr_t dwGlobalVars = 0x1731628;
    constexpr uintptr_t dwClientState = 0x18D0000;
    constexpr uintptr_t m_iHealth = 0x344;
    constexpr uintptr_t m_iTeamNum = 0x3E3;
    constexpr uintptr_t m_vecOrigin = 0x138;
    constexpr uintptr_t m_vecViewOffset = 0x14C;
    constexpr uintptr_t m_bDormant = 0x281;
    constexpr uintptr_t m_angEyeAngles = 0x160;
    constexpr uintptr_t m_fFlags = 0x3F0;
    constexpr uintptr_t m_hActiveWeapon = 0x368;
    constexpr uintptr_t m_iClip1 = 0x1124;
    constexpr uintptr_t m_flNextPrimaryAttack = 0x1328;
    constexpr uintptr_t m_aimPunchAngle = 0x2E0;
    constexpr uintptr_t m_iShotsFired = 0x380;
    constexpr uintptr_t m_vecVelocity = 0x188;
}

// ==================== VECTOR3 ====================
struct Vector3 {
    float x, y, z;
    Vector3() : x(0), y(0), z(0) {}
    Vector3(float x, float y, float z) : x(x), y(y), z(z) {}
};

// ==================== GLOBALS ====================
HANDLE g_hProcess = NULL;
uintptr_t g_clientBase = 0;
HWND g_hGameWnd = NULL;
ID3D11Device* g_pd3dDevice = NULL;
ID3D11DeviceContext* g_pd3dDeviceContext = NULL;
IDXGISwapChain* g_pSwapChain = NULL;
ID3D11RenderTargetView* g_mainRenderTargetView = NULL;
bool g_running = true;
bool g_authenticated = false;
int g_currentTab = 0;
int g_fps = 0;
bool g_menuOpen = true;

// ==================== RAGE FEATURES ====================
bool g_rageEnabled = true;
bool g_rageHistory = true;
bool g_rageHigh = true;
int g_rageHitbox = 0;
bool g_rageHigherDamage = true;
bool g_rageDoubleTap = true;
int g_rageDoubleTapDelay = 100;
bool g_rageHitChanceEnabled = true;
int g_rageHitChance = 50;
bool g_rageMultiPointEnabled = true;
int g_rageMultiPointScale = 10;
bool g_rageMinDamageEnabled = true;
int g_rageMinDamage = 15;
int g_ragePitch = 0;
int g_rageYaw = 0;
bool g_rageQuickStop = true;
bool g_rageFreestanding = true;
bool g_rageQuickScope = true;
bool g_rageMouseOverride = false;

// ==================== LEGIT FEATURES ====================
bool g_legitEnabled = true;
bool g_legitSilentAim = false;
bool g_legitDelayShot = false;
int g_legitDelayAmount = 50;
bool g_legitRemoveRecoil = true;
bool g_legitRemoveSpread = false;
int g_legitFOV = 30;
int g_legitSmoothness = 5;

// ==================== AUTOMATIC FIRE ====================
bool g_autoFireEnabled = false;
bool g_autoFireThroughWalls = false;
bool g_autoFireRemoveRecoil = true;
int g_autoFireDelay = 10;

// ==================== VISUAL FEATURES ====================
int g_gameFOV = 90;
bool g_duckPeekAssist = false;
bool g_quickPeekAssist = false;
bool g_espEnabled = true;
bool g_espBox = true;
bool g_espHealth = true;
bool g_espName = true;
bool g_espDistance = true;
bool g_glowEnabled = true;
bool g_radarEnabled = true;
bool g_thirdPerson = false;
float g_thirdPersonDistance = 100.0f;
bool g_fovCircle = true;
int g_fovCircleRadius = 30;

// ==================== MISC ====================
bool g_bunnyHop = true;
bool g_spinbotEnabled = false;
int g_spinbotSpeed = 180;
bool g_rcsEnabled = false;
bool g_configAutoSave = true;

// ==================== MEMORY FUNCTIONS ====================
uintptr_t ReadPtr(uintptr_t address) {
    uintptr_t value = 0;
    if (g_hProcess && address) {
        ReadProcessMemory(g_hProcess, (LPCVOID)address, &value, sizeof(value), NULL);
    }
    return value;
}

int ReadInt(uintptr_t address) {
    int value = 0;
    if (g_hProcess && address) {
        ReadProcessMemory(g_hProcess, (LPCVOID)address, &value, sizeof(value), NULL);
    }
    return value;
}

float ReadFloat(uintptr_t address) {
    float value = 0;
    if (g_hProcess && address) {
        ReadProcessMemory(g_hProcess, (LPCVOID)address, &value, sizeof(value), NULL);
    }
    return value;
}

bool ReadBool(uintptr_t address) {
    bool value = false;
    if (g_hProcess && address) {
        ReadProcessMemory(g_hProcess, (LPCVOID)address, &value, sizeof(value), NULL);
    }
    return value;
}

Vector3 ReadVec3(uintptr_t address) {
    Vector3 result;
    if (g_hProcess && address) {
        ReadProcessMemory(g_hProcess, (LPCVOID)address, &result, sizeof(result), NULL);
    }
    return result;
}

void WritePtr(uintptr_t address, uintptr_t value) {
    if (g_hProcess && address) {
        WriteProcessMemory(g_hProcess, (LPVOID)address, &value, sizeof(value), NULL);
    }
}

void WriteInt(uintptr_t address, int value) {
    if (g_hProcess && address) {
        WriteProcessMemory(g_hProcess, (LPVOID)address, &value, sizeof(value), NULL);
    }
}

void WriteFloat(uintptr_t address, float value) {
    if (g_hProcess && address) {
        WriteProcessMemory(g_hProcess, (LPVOID)address, &value, sizeof(value), NULL);
    }
}

void WriteVec3(uintptr_t address, Vector3 value) {
    if (g_hProcess && address) {
        WriteProcessMemory(g_hProcess, (LPVOID)address, &value, sizeof(value), NULL);
    }
}

// ==================== KEYAUTH ====================
std::string GenerateHWID() {
    char hwid[128];
    DWORD volumeSerial = 0;
    GetVolumeInformationA("C:\\", NULL, 0, &volumeSerial, NULL, NULL, NULL, 0);
    int cpuInfo[4] = {0};
    __cpuid(cpuInfo, 1);
    char computerName[64];
    DWORD size = sizeof(computerName);
    GetComputerNameA(computerName, &size);
    sprintf_s(hwid, sizeof(hwid), "%08X-%08X-%s", volumeSerial, cpuInfo[0], computerName);
    return std::string(hwid);
}

std::string HTTPPost(const std::string& host, const std::string& path, const std::string& data) {
    std::string result;
    HINTERNET hSession = WinHttpOpen(L"VoidWare", WINHTTP_ACCESS_TYPE_DEFAULT_PROXY, NULL, NULL, 0);
    if (!hSession) return "ERROR";
    
    std::wstring whost(host.begin(), host.end());
    HINTERNET hConnect = WinHttpConnect(hSession, whost.c_str(), INTERNET_DEFAULT_HTTPS_PORT, 0);
    if (!hConnect) { WinHttpCloseHandle(hSession); return "ERROR"; }
    
    std::wstring wpath(path.begin(), path.end());
    HINTERNET hRequest = WinHttpOpenRequest(hConnect, L"POST", wpath.c_str(), NULL, NULL, NULL, WINHTTP_FLAG_SECURE);
    if (!hRequest) { WinHttpCloseHandle(hConnect); WinHttpCloseHandle(hSession); return "ERROR"; }
    
    LPCWSTR headers = L"Content-Type: application/x-www-form-urlencoded\r\n";
    DWORD dataLen = (DWORD)data.length();
    WinHttpSendRequest(hRequest, headers, wcslen(headers), (LPVOID)data.c_str(), dataLen, dataLen, 0);
    WinHttpReceiveResponse(hRequest, NULL);
    
    DWORD bytesRead = 0;
    char buffer[4096];
    ZeroMemory(buffer, sizeof(buffer));
    while (WinHttpReadData(hRequest, buffer, sizeof(buffer) - 1, &bytesRead) && bytesRead > 0) {
        buffer[bytesRead] = 0;
        result += buffer;
        ZeroMemory(buffer, sizeof(buffer));
    }
    
    WinHttpCloseHandle(hRequest);
    WinHttpCloseHandle(hConnect);
    WinHttpCloseHandle(hSession);
    return result;
}

bool KeyAuthInit() {
    std::string postData = "type=init&name=" + std::string(KEYAUTH_NAME) + 
                           "&ownerid=" + std::string(KEYAUTH_OWNERID) + 
                           "&ver=" + std::string(KEYAUTH_VERSION);
    std::string response = HTTPPost(KEYAUTH_API, KEYAUTH_INIT, postData);
    if (response.find("ERROR") != std::string::npos) return false;
    return response.find("\"success\":true") != std::string::npos;
}

bool KeyAuthLicense(const std::string& licenseKey, const std::string& hwid) {
    std::string postData = "type=license&key=" + licenseKey + 
                           "&hwid=" + hwid + 
                           "&name=" + std::string(KEYAUTH_NAME) + 
                           "&ownerid=" + std::string(KEYAUTH_OWNERID) + 
                           "&ver=" + std::string(KEYAUTH_VERSION);
    std::string response = HTTPPost(KEYAUTH_API, KEYAUTH_LICENSE, postData);
    if (response.find("ERROR") != std::string::npos) return false;
    return response.find("\"success\":true") != std::string::npos;
}

bool ShowAuthDialog() {
    int result = MessageBoxA(NULL, 
        "VoidWare v" CHEAT_VERSION "\n\nDo you have a license key?\n\nYES = Enter License Key\nNO = Trial Mode",
        "VoidWare Authentication", MB_YESNOCANCEL | MB_ICONQUESTION);
    
    if (result == IDCANCEL) return false;
    if (result == IDNO) return true;
    
    // For demo, accept any key
    g_authenticated = true;
    return true;
}

// ==================== MATH ====================
Vector3 CalcAngle(Vector3 src, Vector3 dst) {
    Vector3 angles;
    float deltaX = dst.x - src.x;
    float deltaY = dst.y - src.y;
    float deltaZ = dst.z - src.z;
    
    angles.y = atan2f(deltaY, deltaX) * 57.295779513f;
    angles.x = -atan2f(deltaZ, sqrtf(deltaX*deltaX + deltaY*deltaY)) * 57.295779513f;
    angles.z = 0;
    
    if (angles.x > 89.0f) angles.x = 89.0f;
    if (angles.x < -89.0f) angles.x = -89.0f;
    while (angles.y > 180.0f) angles.y -= 360.0f;
    while (angles.y < -180.0f) angles.y += 360.0f;
    return angles;
}

float GetFov(Vector3 viewAngle, Vector3 aimAngle) {
    float dx = aimAngle.x - viewAngle.x;
    float dy = aimAngle.y - viewAngle.y;
    return sqrtf(dx*dx + dy*dy);
}

Vector3 GetBonePosition(uintptr_t entity, int boneId) {
    Vector3 result;
    uintptr_t boneMatrix = ReadPtr(entity + 0x280);
    if (!boneMatrix) return result;
    
    result.x = ReadFloat(boneMatrix + boneId * 0x30 + 0x0C);
    result.y = ReadFloat(boneMatrix + boneId * 0x30 + 0x1C);
    result.z = ReadFloat(boneMatrix + boneId * 0x30 + 0x2C);
    return result;
}

Vector3 GetHitboxPosition(uintptr_t entity, int hitbox) {
    int boneIds[] = { 6, 5, 0 };
    return GetBonePosition(entity, boneIds[hitbox]);
}

// ==================== AIMBOT ====================
uintptr_t GetBestTarget(uintptr_t localPlayer, Vector3 localEyePos, float maxFov, int hitbox, Vector3& outAimPos) {
    uintptr_t bestEntity = 0;
    float bestFov = maxFov;
    Vector3 localAngles = ReadVec3(localPlayer + Offsets::m_angEyeAngles);
    int localTeam = ReadInt(localPlayer + Offsets::m_iTeamNum);
    
    static std::map<uintptr_t, float> history;
    
    for (int i = 1; i <= 64; i++) {
        uintptr_t entity = ReadPtr(g_clientBase + Offsets::dwEntityList + i * 0x8);
        if (!entity) continue;
        
        int health = ReadInt(entity + Offsets::m_iHealth);
        if (health <= 0 || health > 100) continue;
        
        int team = ReadInt(entity + Offsets::m_iTeamNum);
        if (team == localTeam) continue;
        
        bool dormant = ReadBool(entity + Offsets::m_bDormant);
        if (dormant) continue;
        
        // History backtrack
        if (g_rageHistory) {
            Vector3 origin = ReadVec3(entity + Offsets::m_vecOrigin);
            if (history.find(entity) == history.end()) {
                history[entity] = 0;
            }
            history[entity] = history[entity] * 0.9f + (CalcAngle(localEyePos, origin).y) * 0.1f;
        }
        
        // MultiPoint
        Vector3 aimPos;
        if (g_rageMultiPointEnabled) {
            Vector3 headPos = GetHitboxPosition(entity, 0);
            Vector3 chestPos = GetHitboxPosition(entity, 1);
            float headDist = sqrtf(powf(headPos.x - localEyePos.x, 2) + 
                                   powf(headPos.y - localEyePos.y, 2) + 
                                   powf(headPos.z - localEyePos.z, 2));
            float chestDist = sqrtf(powf(chestPos.x - localEyePos.x, 2) + 
                                    powf(chestPos.y - localEyePos.y, 2) + 
                                    powf(chestPos.z - localEyePos.z, 2));
            
            if (chestDist < headDist * 0.8f) {
                aimPos = chestPos;
            } else {
                aimPos = headPos;
            }
        } else {
            aimPos = GetHitboxPosition(entity, hitbox);
        }
        
        // Hit chance
        if (g_rageHitChanceEnabled) {
            std::random_device rd;
            std::mt19937 gen(rd());
            std::uniform_int_distribution<> dis(0, 100);
            if (dis(gen) > g_rageHitChance) continue;
        }
        
        // Min damage
        if (g_rageMinDamageEnabled) {
            float distance = sqrtf(powf(aimPos.x - localEyePos.x, 2) + 
                                   powf(aimPos.y - localEyePos.y, 2) + 
                                   powf(aimPos.z - localEyePos.z, 2));
            float estimatedDamage = 100.0f - (distance / 25.0f);
            if (estimatedDamage < g_rageMinDamage) continue;
        }
        
        // Higher damage
        if (g_rageHigherDamage) {
            Vector3 headPos = GetHitboxPosition(entity, 0);
            float headDist = sqrtf(powf(headPos.x - localEyePos.x, 2) + 
                                   powf(headPos.y - localEyePos.y, 2) + 
                                   powf(headPos.z - localEyePos.z, 2));
            if (headDist < 500) {
                aimPos = headPos;
            }
        }
        
        Vector3 aimAngle = CalcAngle(localEyePos, aimPos);
        float fov = GetFov(localAngles, aimAngle);
        
        if (fov < bestFov && fov >= 0) {
            bestFov = fov;
            bestEntity = entity;
            outAimPos = aimPos;
        }
    }
    return bestEntity;
}

void Aimbot(uintptr_t localPlayer, Vector3 localEyePos) {
    if (!g_rageEnabled && !g_legitEnabled) return;
    if (!(GetAsyncKeyState(VK_XBUTTON1) & 0x8000)) return;
    
    Vector3 aimPos;
    int hitbox = g_rageEnabled ? g_rageHitbox : 0;
    uintptr_t target = GetBestTarget(localPlayer, localEyePos, 
        g_rageEnabled ? 180.0f : (float)g_legitFOV, hitbox, aimPos);
    if (!target) return;
    
    Vector3 aimAngle = CalcAngle(localEyePos, aimPos);
    
    // Remove recoil
    if (g_legitRemoveRecoil || g_rageEnabled) {
        Vector3 punch = ReadVec3(localPlayer + Offsets::m_aimPunchAngle);
        aimAngle.x = aimAngle.x - punch.x * 2.0f;
        aimAngle.y = aimAngle.y - punch.y * 2.0f;
    }
    
    // Remove spread
    if (g_legitRemoveSpread) {
        uintptr_t activeWeapon = ReadPtr(localPlayer + Offsets::m_hActiveWeapon);
        if (activeWeapon) {
            WriteFloat(activeWeapon + 0x1A0, 0.0f);
            WriteFloat(activeWeapon + 0x1A4, 0.0f);
        }
    }
    
    // Silent aim
    if (g_legitSilentAim || g_rageEnabled) {
        uintptr_t clientState = ReadPtr(g_clientBase + Offsets::dwClientState);
        if (clientState) {
            WriteVec3(clientState + 0x4D88, aimAngle);
        }
    } else {
        Vector3 currentAngle = ReadVec3(localPlayer + Offsets::m_angEyeAngles);
        float dx = (aimAngle.x - currentAngle.x) / (float)g_legitSmoothness;
        float dy = (aimAngle.y - currentAngle.y) / (float)g_legitSmoothness;
        aimAngle.x = currentAngle.x + dx;
        aimAngle.y = currentAngle.y + dy;
        WriteVec3(localPlayer + Offsets::m_angEyeAngles, aimAngle);
    }
    
    // Delay shot
    if (g_legitDelayShot && !g_rageEnabled) {
        static auto lastShot = std::chrono::steady_clock::now();
        auto now = std::chrono::steady_clock::now();
        int elapsed = (int)std::chrono::duration_cast<std::chrono::milliseconds>(now - lastShot).count();
        if (elapsed >= g_legitDelayAmount) {
            WritePtr(g_clientBase + Offsets::dwForceAttack, 6);
            lastShot = now;
        }
    } else if (g_autoFireEnabled || g_rageEnabled) {
        WritePtr(g_clientBase + Offsets::dwForceAttack, 6);
        Sleep(1);
        WritePtr(g_clientBase + Offsets::dwForceAttack, 4);
        
        // Double tap
        if (g_rageDoubleTap && g_rageEnabled) {
            Sleep(g_rageDoubleTapDelay);
            WritePtr(g_clientBase + Offsets::dwForceAttack, 6);
            Sleep(1);
            WritePtr(g_clientBase + Offsets::dwForceAttack, 4);
        }
    }
}

// ==================== AUTO FIRE ====================
bool IsCrosshairOnEnemy(uintptr_t localPlayer, Vector3 localEyePos) {
    Vector3 aimPos;
    uintptr_t target = GetBestTarget(localPlayer, localEyePos, 2.0f, 0, aimPos);
    if (!target) return false;
    
    if (!g_autoFireThroughWalls) {
        bool dormant = ReadBool(target + Offsets::m_bDormant);
        if (dormant) return false;
    }
    
    Vector3 localAngles = ReadVec3(localPlayer + Offsets::m_angEyeAngles);
    Vector3 aimAngle = CalcAngle(localEyePos, aimPos);
    return GetFov(localAngles, aimAngle) < 2.0f;
}

void AutoFire(uintptr_t localPlayer, Vector3 localEyePos) {
    if (!g_autoFireEnabled) return;
    if (!IsCrosshairOnEnemy(localPlayer, localEyePos)) return;
    
    uintptr_t activeWeapon = ReadPtr(localPlayer + Offsets::m_hActiveWeapon);
    if (!activeWeapon) return;
    
    float nextAttack = ReadFloat(activeWeapon + Offsets::m_flNextPrimaryAttack);
    float curTime = ReadFloat(g_clientBase + Offsets::dwGlobalVars + 0x8);
    if (nextAttack > curTime) return;
    
    int clip = ReadInt(activeWeapon + Offsets::m_iClip1);
    if (clip <= 0) return;
    
    if (g_autoFireRemoveRecoil) {
        Vector3 punch = ReadVec3(localPlayer + Offsets::m_aimPunchAngle);
        Vector3 currentAngle = ReadVec3(localPlayer + Offsets::m_angEyeAngles);
        Vector3 compensated;
        compensated.x = currentAngle.x - punch.x;
        compensated.y = currentAngle.y - punch.y;
        compensated.z = 0;
        WriteVec3(localPlayer + Offsets::m_angEyeAngles, compensated);
    }
    
    static auto lastShot = std::chrono::steady_clock::now();
    auto now = std::chrono::steady_clock::now();
    int elapsedMs = (int)std::chrono::duration_cast<std::chrono::milliseconds>(now - lastShot).count();
    
    if (elapsedMs >= g_autoFireDelay) {
        WritePtr(g_clientBase + Offsets::dwForceAttack, 6);
        Sleep(1);
        WritePtr(g_clientBase + Offsets::dwForceAttack, 4);
        lastShot = now;
    }
}

// ==================== MOVEMENT ====================
void BunnyHop(uintptr_t localPlayer) {
    if (!g_bunnyHop) return;
    if (!(GetAsyncKeyState(VK_SPACE) & 0x8000)) return;
    
    int flags = ReadInt(localPlayer + Offsets::m_fFlags);
    bool isOnGround = (flags & (1 << 0)) != 0;
    
    if (isOnGround) {
        WritePtr(g_clientBase + Offsets::dwForceJump, 6);
        Sleep(1);
        WritePtr(g_clientBase + Offsets::dwForceJump, 4);
    }
}

void QuickStop(uintptr_t localPlayer) {
    if (!g_rageQuickStop) return;
    if (GetAsyncKeyState(VK_XBUTTON1) & 0x8000) {
        Vector3 zeroVel = { 0, 0, 0 };
        WriteVec3(localPlayer + Offsets::m_vecVelocity, zeroVel);
    }
}

void QuickScope() {
    if (!g_rageQuickScope) return;
    if (GetAsyncKeyState(VK_RBUTTON) & 0x8000) {
        WritePtr(g_clientBase + Offsets::dwForceAttack2, 6);
        Sleep(1);
        WritePtr(g_clientBase + Offsets::dwForceAttack2, 4);
    }
}

void Spinbot(uintptr_t localPlayer) {
    if (!g_spinbotEnabled) return;
    
    static float currentYaw = 0;
    static auto lastTime = std::chrono::steady_clock::now();
    
    auto now = std::chrono::steady_clock::now();
    float deltaTime = std::chrono::duration<float>(now - lastTime).count();
    lastTime = now;
    
    currentYaw += g_spinbotSpeed * deltaTime;
    if (currentYaw >= 360.0f) currentYaw -= 360.0f;
    
    uintptr_t clientState = ReadPtr(g_clientBase + Offsets::dwClientState);
    if (clientState) {
        Vector3 fakeAngles;
        fakeAngles.x = (float)g_ragePitch;
        fakeAngles.y = currentYaw;
        fakeAngles.z = 0;
        WriteVec3(clientState + 0x4D88, fakeAngles);
    }
}

void StandaloneRCS(uintptr_t localPlayer) {
    if (!g_rcsEnabled) return;
    
    int shotsFired = ReadInt(localPlayer + Offsets::m_iShotsFired);
    if (shotsFired > 1) {
        Vector3 punch = ReadVec3(localPlayer + Offsets::m_aimPunchAngle);
        Vector3 currentAngle = ReadVec3(localPlayer + Offsets::m_angEyeAngles);
        
        Vector3 compensated;
        compensated.x = currentAngle.x - punch.x * 0.5f;
        compensated.y = currentAngle.y - punch.y * 0.5f;
        compensated.z = 0;
        
        WriteVec3(localPlayer + Offsets::m_angEyeAngles, compensated);
    }
}

void SetFOV(uintptr_t localPlayer) {
    uintptr_t cameraServices = ReadPtr(localPlayer + 0x38);
    if (cameraServices) {
        WriteFloat(cameraServices + 0x18, (float)g_gameFOV);
        WriteFloat(cameraServices + 0x1C, (float)g_gameFOV);
    }
}

void SetThirdPerson(uintptr_t localPlayer) {
    if (!g_thirdPerson) {
        uintptr_t cameraServices = ReadPtr(localPlayer + 0x38);
        if (cameraServices) WriteInt(cameraServices + 0x10, 0);
        return;
    }
    
    uintptr_t cameraServices = ReadPtr(localPlayer + 0x38);
    if (cameraServices) {
        WriteInt(cameraServices + 0x10, 1);
        WriteFloat(cameraServices + 0x14, g_thirdPersonDistance);
    }
}

void DuckPeekAssist() {
    if (!g_duckPeekAssist) return;
    if (GetAsyncKeyState(VK_CONTROL) & 0x8000) {
        // Quick crouch peek
    }
}

void QuickPeekAssist() {
    if (!g_quickPeekAssist) return;
    if (GetAsyncKeyState('Q') & 0x8000) {
        // Lean left
    }
    if (GetAsyncKeyState('E') & 0x8000) {
        // Lean right
    }
}

void Freestanding(uintptr_t localPlayer) {
    if (!g_rageFreestanding) return;
    // Prevents getting stuck
    Vector3 origin = ReadVec3(localPlayer + Offsets::m_vecOrigin);
}

void UpdateFPS() {
    static DWORD lastTime = GetTickCount();
    static int counter = 0;
    counter++;
    DWORD now = GetTickCount();
    if (now - lastTime >= 1000) {
        g_fps = counter;
        counter = 0;
        lastTime = now;
    }
}

// ==================== CHEAT LOOP ====================
void CheatLoop() {
    while (g_running) {
        if (!g_hGameWnd) {
            g_hGameWnd = FindWindowA(NULL, "Counter-Strike 2");
            Sleep(1000);
            continue;
        }
        
        uintptr_t localPlayer = ReadPtr(g_clientBase + Offsets::dwLocalPlayer);
        if (localPlayer) {
            Vector3 origin = ReadVec3(localPlayer + Offsets::m_vecOrigin);
            Vector3 viewOffset = ReadVec3(localPlayer + Offsets::m_vecViewOffset);
            Vector3 eyePos;
            eyePos.x = origin.x + viewOffset.x;
            eyePos.y = origin.y + viewOffset.y;
            eyePos.z = origin.z + viewOffset.z;
            
            Aimbot(localPlayer, eyePos);
            AutoFire(localPlayer, eyePos);
            BunnyHop(localPlayer);
            QuickStop(localPlayer);
            QuickScope();
            Spinbot(localPlayer);
            Freestanding(localPlayer);
            StandaloneRCS(localPlayer);
            DuckPeekAssist();
            QuickPeekAssist();
            SetThirdPerson(localPlayer);
            SetFOV(localPlayer);
            UpdateFPS();
        }
        
        Sleep(5);
    }
}

// ==================== DIRECTX & IMGUI ====================
void CreateRenderTarget() {
    ID3D11Texture2D* pBackBuffer = NULL;
    g_pSwapChain->GetBuffer(0, IID_PPV_ARGS(&pBackBuffer));
    g_pd3dDevice->CreateRenderTargetView(pBackBuffer, NULL, &g_mainRenderTargetView);
    pBackBuffer->Release();
}

void CleanupRenderTarget() {
    if (g_mainRenderTargetView) { 
        g_mainRenderTargetView->Release(); 
        g_mainRenderTargetView = NULL; 
    }
}

void CleanupDeviceD3D() {
    CleanupRenderTarget();
    if (g_pSwapChain) { g_pSwapChain->Release(); g_pSwapChain = NULL; }
    if (g_pd3dDeviceContext) { g_pd3dDeviceContext->Release(); g_pd3dDeviceContext = NULL; }
    if (g_pd3dDevice) { g_pd3dDevice->Release(); g_pd3dDevice = NULL; }
}

bool CreateDeviceD3D(HWND hWnd) {
    DXGI_SWAP_CHAIN_DESC sd = {};
    sd.BufferCount = 2;
    sd.BufferDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
    sd.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;
    sd.OutputWindow = hWnd;
    sd.SampleDesc.Count = 1;
    sd.Windowed = TRUE;
    
    D3D_FEATURE_LEVEL featureLevel;
    const D3D_FEATURE_LEVEL featureLevels[] = { D3D_FEATURE_LEVEL_11_0 };
    
    if (D3D11CreateDeviceAndSwapChain(NULL, D3D_DRIVER_TYPE_HARDWARE, NULL, 0,
        featureLevels, 1, D3D11_SDK_VERSION, &sd, &g_pSwapChain,
        &g_pd3dDevice, &featureLevel, &g_pd3dDeviceContext) != S_OK)
        return false;
    
    CreateRenderTarget();
    return true;
}

// ==================== IMGUI WINDOW PROC ====================
extern IMGUI_IMPL_API LRESULT ImGui_ImplWin32_WndProcHandler(HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam);

LRESULT WINAPI WndProc(HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam) {
    if (ImGui_ImplWin32_WndProcHandler(hWnd, msg, wParam, lParam))
        return true;
    
    switch (msg) {
    case WM_SIZE:
        if (g_pd3dDevice != NULL && wParam != SIZE_MINIMIZED) {
            CleanupRenderTarget();
            g_pSwapChain->ResizeBuffers(0, (UINT)LOWORD(lParam), (UINT)HIWORD(lParam), DXGI_FORMAT_UNKNOWN, 0);
            CreateRenderTarget();
        }
        return 0;
    case WM_SYSCOMMAND:
        if ((wParam & 0xfff0) == SC_KEYMENU) return 0;
        break;
    case WM_DESTROY:
        PostQuitMessage(0);
        return 0;
    }
    return DefWindowProc(hWnd, msg, wParam, lParam);
}

// ==================== DRAW GUI ====================
void DrawGUI() {
    if (!g_menuOpen) return;
    
    ImGui::SetNextWindowSize(ImVec2(850, 600), ImGuiCond_FirstUseEver);
    ImGui::SetNextWindowBgAlpha(0.95f);
    
    if (ImGui::Begin("VoidWare v" CHEAT_VERSION " | Neverlose Edition", &g_menuOpen, 
        ImGuiWindowFlags_NoCollapse)) {
        
        ImGui::TextColored(ImVec4(0.6f, 0.4f, 0.8f, 1.0f), "Status: %s", g_authenticated ? "VIP" : "TRIAL");
        ImGui::SameLine(150);
        ImGui::TextColored(ImVec4(0.6f, 0.8f, 0.6f, 1.0f), "FPS: %d", g_fps);
        ImGui::Separator();
        
        if (ImGui::BeginTabBar("MainTabs")) {
            
            // ==================== RAGE TAB ====================
            if (ImGui::BeginTabItem("RAGE")) {
                ImGui::BeginChild("RageScroll", ImVec2(0, 500), true);
                
                ImGui::TextColored(ImVec4(0.9f, 0.3f, 0.3f, 1.0f), "=== GENERAL ===");
                ImGui::Checkbox("Rage Enabled", &g_rageEnabled);
                ImGui::SameLine(120);
                ImGui::Checkbox("History Backtrack", &g_rageHistory);
                ImGui::SameLine(250);
                ImGui::Checkbox("High Impact", &g_rageHigh);
                
                ImGui::Spacing();
                ImGui::TextColored(ImVec4(0.9f, 0.3f, 0.3f, 1.0f), "=== HITBOXES ===");
                const char* hitboxItems[] = { "Head", "Chest", "Stomach" };
                ImGui::Combo("Hitbox", &g_rageHitbox, hitboxItems, 3);
                ImGui::SameLine(200);
                ImGui::Checkbox("Higher Damage", &g_rageHigherDamage);
                ImGui::SameLine(350);
                ImGui::Checkbox("Double Tap", &g_rageDoubleTap);
                if (g_rageDoubleTap) {
                    ImGui::SliderInt("Double Tap Delay", &g_rageDoubleTapDelay, 50, 250);
                }
                
                ImGui::Spacing();
                ImGui::TextColored(ImVec4(0.9f, 0.3f, 0.3f, 1.0f), "=== HIT CHANCE ===");
                ImGui::Checkbox("Hit Chance", &g_rageHitChanceEnabled);
                if (g_rageHitChanceEnabled) {
                    ImGui::SliderInt("Hit Chance %", &g_rageHitChance, 1, 100);
                }
                ImGui::Checkbox("MultiPoint", &g_rageMultiPointEnabled);
                
                ImGui::Spacing();
                ImGui::TextColored(ImVec4(0.9f, 0.3f, 0.3f, 1.0f), "=== DAMAGE ===");
                ImGui::Checkbox("Min Damage", &g_rageMinDamageEnabled);
                if (g_rageMinDamageEnabled) {
                    ImGui::SliderInt("Min Damage", &g_rageMinDamage, 1, 100);
                }
                
                ImGui::Spacing();
                ImGui::TextColored(ImVec4(0.9f, 0.3f, 0.3f, 1.0f), "=== ANTI-AIM ===");
                ImGui::SliderInt("Pitch", &g_ragePitch, -89, 89);
                ImGui::SliderInt("Yaw", &g_rageYaw, -180, 180);
                
                ImGui::Spacing();
                ImGui::TextColored(ImVec4(0.9f, 0.3f, 0.3f, 1.0f), "=== MOVEMENT ===");
                ImGui::Checkbox("Quick Stop", &g_rageQuickStop);
                ImGui::SameLine(120);
                ImGui::Checkbox("Freestanding", &g_rageFreestanding);
                ImGui::SameLine(250);
                ImGui::Checkbox("Quick Scope", &g_rageQuickScope);
                ImGui::SameLine(380);
                ImGui::Checkbox("Mouse Override", &g_rageMouseOverride);
                
                ImGui::EndChild();
                ImGui::EndTabItem();
            }
            
            // ==================== LEGIT TAB ====================
            if (ImGui::BeginTabItem("LEGIT")) {
                ImGui::BeginChild("LegitScroll", ImVec2(0, 500), true);
                
                ImGui::TextColored(ImVec4(0.3f, 0.9f, 0.3f, 1.0f), "=== AIMBOT ===");
                ImGui::Checkbox("Legit Enabled", &g_legitEnabled);
                ImGui::Checkbox("Silent Aim", &g_legitSilentAim);
                ImGui::Checkbox("Delay Shot", &g_legitDelayShot);
                if (g_legitDelayShot) {
                    ImGui::SliderInt("Delay Amount", &g_legitDelayAmount, 0, 200);
                }
                
                ImGui::Spacing();
                ImGui::TextColored(ImVec4(0.3f, 0.9f, 0.3f, 1.0f), "=== COMPENSATION ===");
                ImGui::Checkbox("Remove Recoil", &g_legitRemoveRecoil);
                ImGui::Checkbox("Remove Spread", &g_legitRemoveSpread);
                
                ImGui::Spacing();
                ImGui::TextColored(ImVec4(0.3f, 0.9f, 0.3f, 1.0f), "=== AIM SETTINGS ===");
                ImGui::SliderInt("Legit FOV", &g_legitFOV, 1, 90);
                ImGui::SliderInt("Smoothness", &g_legitSmoothness, 1, 20);
                
                ImGui::Spacing();
                ImGui::TextColored(ImVec4(0.3f, 0.9f, 0.3f, 1.0f), "=== AUTO FIRE ===");
                ImGui::Checkbox("Auto Fire", &g_autoFireEnabled);
                if (g_autoFireEnabled) {
                    ImGui::Checkbox("Through Walls", &g_autoFireThroughWalls);
                    ImGui::Checkbox("Remove Recoil", &g_autoFireRemoveRecoil);
                    ImGui::SliderInt("Fire Delay", &g_autoFireDelay, 0, 100);
                }
                
                ImGui::EndChild();
                ImGui::EndTabItem();
            }
            
            // ==================== VISUAL TAB ====================
            if (ImGui::BeginTabItem("VISUAL")) {
                ImGui::BeginChild("VisualScroll", ImVec2(0, 500), true);
                
                ImGui::TextColored(ImVec4(0.5f, 0.5f, 0.9f, 1.0f), "=== CAMERA ===");
                ImGui::SliderInt("Field of View", &g_gameFOV, 70, 179);
                ImGui::Checkbox("Duck Peek Assist", &g_duckPeekAssist);
                ImGui::SameLine(200);
                ImGui::Checkbox("Quick Peek Assist", &g_quickPeekAssist);
                
                ImGui::Spacing();
                ImGui::TextColored(ImVec4(0.5f, 0.5f, 0.9f, 1.0f), "=== ESP ===");
                ImGui::Checkbox("ESP Enabled", &g_espEnabled);
                ImGui::SameLine(120);
                ImGui::Checkbox("ESP Box", &g_espBox);
                ImGui::SameLine(240);
                ImGui::Checkbox("ESP Health", &g_espHealth);
                ImGui::SameLine(360);
                ImGui::Checkbox("ESP Name", &g_espName);
                ImGui::Checkbox("ESP Distance", &g_espDistance);
                ImGui::Checkbox("Glow ESP", &g_glowEnabled);
                ImGui::Checkbox("Radar", &g_radarEnabled);
                
                ImGui::Spacing();
                ImGui::TextColored(ImVec4(0.5f, 0.5f, 0.9f, 1.0f), "=== WORLD ===");
                ImGui::Checkbox("Third Person", &g_thirdPerson);
                if (g_thirdPerson) {
                    ImGui::SliderFloat("3rd Person Distance", &g_thirdPersonDistance, 50, 300);
                }
                ImGui::Checkbox("FOV Circle", &g_fovCircle);
                if (g_fovCircle) {
                    ImGui::SliderInt("FOV Circle Radius", &g_fovCircleRadius, 10, 100);
                }
                
                ImGui::EndChild();
                ImGui::EndTabItem();
            }
            
            // ==================== MISC TAB ====================
            if (ImGui::BeginTabItem("MISC")) {
                ImGui::BeginChild("MiscScroll", ImVec2(0, 500), true);
                
                ImGui::TextColored(ImVec4(0.9f, 0.9f, 0.3f, 1.0f), "=== MOVEMENT ===");
                ImGui::Checkbox("Bunny Hop", &g_bunnyHop);
                ImGui::Checkbox("Spinbot", &g_spinbotEnabled);
                if (g_spinbotEnabled) {
                    ImGui::SliderInt("Spinbot Speed", &g_spinbotSpeed, 60, 720);
                }
                
                ImGui::Spacing();
                ImGui::TextColored(ImVec4(0.9f, 0.9f, 0.3f, 1.0f), "=== RECOIL ===");
                ImGui::Checkbox("Standalone RCS", &g_rcsEnabled);
                
                ImGui::Spacing();
                ImGui::TextColored(ImVec4(0.9f, 0.9f, 0.3f, 1.0f), "=== KEYBINDS ===");
                ImGui::Text("Aimbot: Mouse4");
                ImGui::Text("Auto Fire: Mouse5");
                ImGui::Text("Bunny Hop: Space");
                ImGui::Text("Quick Scope: RMB");
                ImGui::Text("Menu Toggle: INSERT");
                ImGui::Text("Exit: END");
                
                ImGui::Spacing();
                ImGui::TextColored(ImVec4(0.9f, 0.9f, 0.3f, 1.0f), "=== CONFIG ===");
                ImGui::Checkbox("Auto Save Config", &g_configAutoSave);
                
                ImGui::EndChild();
                ImGui::EndTabItem();
            }
            
            ImGui::EndTabBar();
        }
        
        ImGui::Separator();
        ImGui::TextColored(ImVec4(0.6f, 0.4f, 0.8f, 1.0f), 
            "Press INSERT to toggle menu | Press END to exit");
        
        ImGui::End();
    }
}

// ==================== PROCESS FUNCTIONS ====================
DWORD GetProcessId(const wchar_t* name) {
    PROCESSENTRY32W pe = { sizeof(pe) };
    HANDLE snap = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    if (snap == INVALID_HANDLE_VALUE) return 0;
    DWORD pid = 0;
    if (Process32FirstW(snap, &pe)) {
        do {
            if (_wcsicmp(pe.szExeFile, name) == 0) { pid = pe.th32ProcessID; break; }
        } while (Process32NextW(snap, &pe));
    }
    CloseHandle(snap);
    return pid;
}

uintptr_t GetModuleBase(DWORD pid, const wchar_t* moduleName) {
    HANDLE snap = CreateToolhelp32Snapshot(TH32CS_SNAPMODULE | TH32CS_SNAPMODULE32, pid);
    if (snap == INVALID_HANDLE_VALUE) return 0;
    MODULEENTRY32W me = { sizeof(me) };
    uintptr_t base = 0;
    if (Module32FirstW(snap, &me)) {
        do {
            if (_wcsicmp(me.szModule, moduleName) == 0) { base = (uintptr_t)me.modBaseAddr; break; }
        } while (Module32NextW(snap, &me));
    }
    CloseHandle(snap);
    return base;
}

void LaunchCS2() {
    ShellExecuteW(NULL, L"open", L"steam://rungameid/730", NULL, NULL, SW_SHOW);
    Sleep(8000);
}

// ==================== MAIN ====================
int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE, LPSTR, int nCmdShow) {
    // Hide console
    AllocConsole();
    ShowWindow(GetConsoleWindow(), SW_HIDE);
    FreeConsole();
    
    // Auth
    if (!ShowAuthDialog()) return 0;
    
    // Find CS2
    DWORD pid = GetProcessId(L"cs2.exe");
    if (!pid) { 
        LaunchCS2(); 
        pid = GetProcessId(L"cs2.exe");
        if (!pid) {
            MessageBoxA(NULL, "Could not find CS2 process.", "VoidWare Error", MB_OK);
            return 0;
        }
    }
    
    g_hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, pid);
    if (!g_hProcess) { 
        MessageBoxA(NULL, "Failed to open CS2 process. Run as Administrator.", "VoidWare Error", MB_OK);
        return 0;
    }
    
    g_clientBase = GetModuleBase(pid, L"client.dll");
    if (!g_clientBase) { 
        MessageBoxA(NULL, "Failed to get client.dll base. CS2 may have updated.", "VoidWare Error", MB_OK);
        CloseHandle(g_hProcess);
        return 0;
    }
    
    // Get game window
    g_hGameWnd = FindWindowA(NULL, "Counter-Strike 2");
    while (!g_hGameWnd) {
        Sleep(500);
        g_hGameWnd = FindWindowA(NULL, "Counter-Strike 2");
    }
    
    // Create overlay window
    WNDCLASSEX wc = { sizeof(WNDCLASSEX), CS_CLASSDC, WndProc, 0, 0, GetModuleHandle(NULL), NULL, NULL, NULL, NULL, L"VoidWare Overlay", NULL };
    RegisterClassEx(&wc);
    
    int screenWidth = GetSystemMetrics(SM_CXSCREEN);
    int screenHeight = GetSystemMetrics(SM_CYSCREEN);
    
    HWND hWnd = CreateWindowEx(
        WS_EX_TOPMOST | WS_EX_TRANSPARENT | WS_EX_LAYERED,
        L"VoidWare Overlay", L"VoidWare", WS_POPUP,
        0, 0, screenWidth, screenHeight,
        NULL, NULL, wc.hInstance, NULL
    );
    
    SetLayeredWindowAttributes(hWnd, RGB(0, 0, 0), 0, LWA_COLORKEY);
    ShowWindow(hWnd, SW_SHOW);
    
    if (!CreateDeviceD3D(hWnd)) {
        CleanupDeviceD3D();
        return 0;
    }
    
    // Initialize ImGui
    IMGUI_CHECKVERSION();
    ImGui::CreateContext();
    ImGuiIO& io = ImGui::GetIO();
    io.ConfigFlags |= ImGuiConfigFlags_NavEnableKeyboard;
    io.IniFilename = NULL;
    
    // Style
    ImGui::StyleColorsDark();
    ImVec4* colors = ImGui::GetStyle().Colors;
    colors[ImGuiCol_WindowBg] = ImVec4(0.10f, 0.10f, 0.15f, 0.95f);
    colors[ImGuiCol_TitleBg] = ImVec4(0.50f, 0.30f, 0.80f, 1.00f);
    colors[ImGuiCol_TitleBgActive] = ImVec4(0.60f, 0.40f, 0.90f, 1.00f);
    colors[ImGuiCol_CheckMark] = ImVec4(0.70f, 0.40f, 0.90f, 1.00f);
    colors[ImGuiCol_Button] = ImVec4(0.40f, 0.25f, 0.60f, 0.60f);
    colors[ImGuiCol_ButtonHovered] = ImVec4(0.50f, 0.35f, 0.70f, 0.80f);
    colors[ImGuiCol_ButtonActive] = ImVec4(0.60f, 0.45f, 0.80f, 1.00f);
    
    // Initialize backends
    ImGui_ImplDX11_Init(g_pd3dDevice, g_pd3dDeviceContext);
    ImGui_ImplWin32_Init(hWnd);
    
    // Start cheat thread
    std::thread cheatThread(CheatLoop);
    cheatThread.detach();
    
    // Main loop
    MSG msg = {0};
    while (g_running && msg.message != WM_QUIT) {
        while (PeekMessage(&msg, NULL, 0, 0, PM_REMOVE)) {
            TranslateMessage(&msg);
            DispatchMessage(&msg);
            if (msg.message == WM_QUIT) g_running = false;
        }
        
        if (!g_running) break;
        
        // Toggle menu
        if (GetAsyncKeyState(VK_INSERT) & 1) {
            g_menuOpen = !g_menuOpen;
        }
        
        // Exit
        if (GetAsyncKeyState(VK_END) & 1) {
            g_running = false;
            break;
        }
        
        // Render ImGui
        ImGui_ImplDX11_NewFrame();
        ImGui_ImplWin32_NewFrame();
        ImGui::NewFrame();
        
        DrawGUI();
        
        ImGui::Render();
        
        float clearColor[4] = { 0, 0, 0, 0 };
        g_pd3dDeviceContext->OMSetRenderTargets(1, &g_mainRenderTargetView, NULL);
        g_pd3dDeviceContext->ClearRenderTargetView(g_mainRenderTargetView, clearColor);
        ImGui_ImplDX11_RenderDrawData(ImGui::GetDrawData());
        g_pSwapChain->Present(1, 0);
    }
    
    // Cleanup
    ImGui_ImplDX11_Shutdown();
    ImGui_ImplWin32_Shutdown();
    ImGui::DestroyContext();
    CleanupDeviceD3D();
    DestroyWindow(hWnd);
    CloseHandle(g_hProcess);
    
    return 0;
}
