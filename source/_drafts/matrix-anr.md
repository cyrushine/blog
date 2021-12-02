TracePlugin.start
SignalAnrTracer.onStartTrace
SignalAnrTracer.onAlive
SignalAnrTracer.nativeInitSignalAnrDetective

static std::optional<AnrDumper> sAnrDumper;
static void nativeInitSignalAnrDetective(JNIEnv *env, jclass, jstring anrTracePath, jstring printTracePath) {
    const char* anrTracePathChar = env->GetStringUTFChars(anrTracePath, nullptr);
    const char* printTracePathChar = env->GetStringUTFChars(printTracePath, nullptr);
    anrTracePathstring = std::string(anrTracePathChar);
    printTracePathstring = std::string(printTracePathChar);
    sAnrDumper.emplace(anrTracePathChar, printTracePathChar, anrDumpCallback);
}

class AnrDumper : public SignalHandler

SignalHandler::SignalHandler() {
    std::lock_guard<std::mutex> lock(sHandlerStackMutex);

    if (!sHandlerStack)
        sHandlerStack = new std::vector<SignalHandler*>;

    installAlternateStackLocked();
    installHandlersLocked();
    sHandlerStack->push_back(this);
}

const int TARGET_SIG = SIGQUIT;
bool SignalHandler::installHandlersLocked() {
    if (sHandlerInstalled) {
        return false;
    }

    if (sigaction(TARGET_SIG, nullptr, &sOldHandlers) == -1) {
        return false;
    }

    struct sigaction sa{};
    sa.sa_sigaction = signalHandler;
    sa.sa_flags = SA_ONSTACK | SA_SIGINFO | SA_RESTART;

    if (sigaction(TARGET_SIG, &sa, nullptr) == -1) {
        ALOGV("Signal handler cannot be installed");
    }

    sHandlerInstalled = true;
    ALOGV("Signal handler installed.");
    return true;
}

void SignalHandler::signalHandler(int sig, siginfo_t* info, void* uc) {
    ALOGV("Entered signal handler.");

    std::unique_lock<std::mutex> lock(sHandlerStackMutex);

    for (auto it = sHandlerStack->rbegin(); it != sHandlerStack->rend(); ++it) {
        (*it)->handleSignal(sig, info, uc);
    }

    lock.unlock();
}

SignalHandler::Result AnrDumper::handleSignal(int sig, const siginfo_t *info, void *uc) {
    // Only process SIGQUIT, which indicates an ANR.
    if (sig != SIGQUIT) return NOT_HANDLED;
    int fromPid1 = info->_si_pad[3];
    int fromPid2 = info->_si_pad[4]; // sender pid
    int myPid = getpid();

    pthread_t thd;
    if (fromPid1 != myPid && fromPid2 != myPid) {
        pthread_create(&thd, nullptr, anrCallback, nullptr);
    } else {
        pthread_create(&thd, nullptr, siUserCallback, nullptr);
    }
    pthread_detach(thd);

    return HANDLED_NO_RETRIGGER;
}

fromPid1 和 fromPid2 是什么？
system         1719    703 9940960 199048 0                   0 S system_server
u10_a369       7414    703 5513524 135784 0                   0 S sample.tencent.matrix

APP 发生 ANR fromPid1:0, fromPid2:1719, myPid:7414
自己发送 SIGQUIT 给自己 fromPid1:0, fromPid2:7414, myPid:7414