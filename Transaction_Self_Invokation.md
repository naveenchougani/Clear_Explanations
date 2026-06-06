
package com.architect.masterclass;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    // Target 1: Bypasses the Proxy completely because it has no annotation
    public void methodA() {
        System.out.println("Executing non-transactional business logic inside methodA...");
        
        // This is implicitly processed by the JVM as: this.methodB();
        methodB(); 
    }

    // Target 2: Transaction boundaries are ignored due to local invocation
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void methodB() {
        System.out.println("Executing database operations inside methodB...");
    }
}

 [ TOMCAT THREAD POOL ]

           |
           v (1) Allocates Native Worker Thread 'Thread-1'. 
           |     JVM allocates a private Memory Stack for Thread-1 to manage stack frames.
           |     Inside the Thread-1 heap object instance, a private 'threadLocals' Map instance 

           |     variable (the "Memory Backpack") is initialized to store thread-isolated data.
      [ Thread-1 ]
           |
           v (2) Executes external call: 'orderService.methodA()' targeting the proxy bean wrapper.
  [ SPRING CGLIB PROXY ] ------------------------------------------------------------------------+

           |                                                                                     |
           +---> (3) Scans internal configuration metadata registry for 'methodA()'.             |

           |     RESULT: No @Transactional annotation detected.                                  |
           +---> (4) SKIPS TransactionManager allocation entirely. Leaves 'threadLocals' empty.  |

           |                                                                                     |
           v (5) Forwards CPU Program Counter (PC) register directly to raw target object RAM address.
 [ RAW OrderService BEAN ] ----------------------------------------------------------------------+

           |                                                                                     |
           +---> (6) Thread-1 pushes a new Stack Frame onto its private Memory Stack to run methodA().

           |     It executes plain, non-transactional business logic inside methodA().          |
           |                                                                                     |
           +---> (7) Encounters 'methodB()' line. JVM executes 'invokevirtual' bytecode instruction.

           |     CRITICAL POINT: This uses the implicit Java 'this' reference.                  |
           |                                                                                     |
           v (8) The CPU Program Counter executes a hardware memory jump straight to methodB().  |
  [ METHOD B BODY ] <----------------------------------------------------------------------------+

           |
           v (9) Thread-1 pushes a new Stack Frame for methodB() onto its private Memory Stack.
           |     Repository query executes. Hibernate needs to stream raw SQL statements.
 [ HIBERNATE ENGINE ]

           |
           v (10) Invokes Spring's Core Infrastructure to search for an active database socket.
 [ TransactionSynchronizationManager ]
           |
           v (11) Triggers 'Thread.currentThread().threadLocals.get(SpringTransactionKey)'.

           |      - 'Thread.currentThread()' queries the physical CPU register to identify Thread-1.
           |      - The JVM uses that pointer to open Thread-1's private 'threadLocals' map instance.
           v
 [ Thread-1 STORAGE BACKPACK ] 

           |
           +---> (12) MAP LOOKUP MECHANICS:
           |     The map is a standard lookup store inside the Thread-1 object structure.
           |     - KEY: 'SpringTransactionKey' (The static DataSource resource identifier object).

           |     - VALUE: Expected to be a 'java.sql.Connection' wrapper object (which holds the
           |       active Operating System TCP socket descriptor to transmit bytes to the DB).
           |
           v (13) RESULT: Map returns NULL (Because the proxy was bypassed at Step 8; no key-value

           |      pair was ever inserted into Thread-1's private map).
           |
           v (14) FALLBACK TRIPPED: Hibernate requests a temporary connection from the HikariCP pool.
   [ HIKARI CP POOL ]

           |
           v (15) Opens the connection's embedded Operating System TCP socket descriptor.
           |      Streams raw query bytes over the network wire.
           v
  [ DATABASE ENGINE ] ---------> (16) CRITICAL FAILURE: No 'START TRANSACTION' packet arrived first.
                                      Engine executes SQL in AUTO-COMMIT MODE.
                                      Changes write permanently to disk with ZERO rollback safety net.



package com.architect.masterclass;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Lazy;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    // Injected CGLIB Proxy reference resolved via the @Lazy helper placeholder system
    @Autowired
    @Lazy
    private OrderService self;

    // Target 1: No transaction annotation, but triggers the self-proxy fix
    public void methodA_Fixed() {
        System.out.println("Executing methodA and forcing an external proxy loop...");
        
        // Explicitly routing through the proxy wrapper object instead of 'this'
        self.methodB(); 
    }

    // Target 2: Transaction boundaries are successfully built on the same thread
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void methodB() {
        System.out.println("Executing core database transactions inside methodB...");
    }
}

 [ TOMCAT THREAD POOL ]


           |
           v (1) Allocates Native Worker Thread 'Thread-1'. 
           |     JVM assigns a private Memory Stack and initializes an empty 'threadLocals' map
           |     (the "Memory Backpack") inside the Thread-1 object structure.
      [ Thread-1 ]

           |
           v (2) Executes external call: 'orderService.methodA_Fixed()' targeting the proxy bean wrapper.
  [ SPRING CGLIB PROXY ] ------------------------------------------------------------------------+
           |                                                                                     |
           +---> (3) Scans internal configuration metadata registry for 'methodA_Fixed()'.       |

           |     RESULT: No @Transactional annotation detected. Proxy steps aside.               |
           v (4) Forwards CPU Program Counter (PC) register directly to raw target object RAM address.
 [ RAW OrderService BEAN ] ----------------------------------------------------------------------+

           |                                                                                     |
           +---> (5) Thread-1 pushes a new Stack Frame onto its private Memory Stack to run methodA_Fixed().

           |     It executes plain, non-transactional business logic.                            |
           |                                                                                     |
           +---> (6) Encounters 'self.methodB()' line. JVM evaluates the 'self' field pointer.

           |     CRITICAL POINT: 'self' points to the synthetic Fake Lazy Proxy placeholder.     |
           |                                                                                     |
           v (7) The CPU Program Counter leaves the raw bean and jumps to the Fake Lazy Proxy.   |
  [ FAKE LAZY PROXY ]

           |
           +---> (8) Wakes up on the first method invocation, reaches into the Spring context container,
           |     extracts the master CGLIB Transactional Proxy, and forwards Thread-1 into it.
           v
  [ MASTER CGLIB PROXY ]

           |
           +---> (9) Interception Engine activates! The proxy scans metadata and reads:
           |     '@Transactional(propagation = Propagation.REQUIRES_NEW)'.
           +---> (10) The proxy pauses execution, requests an active connection ('Connection #1') 

           |     from the HikariCP pool, and sets 'connection1.setAutoCommit(false)' over the TCP wire.
           +---> (11) CORE BINDING ACT: The proxy invokes 'TransactionSynchronizationManager.bindResource()'.
           |     - JVM uses Thread.currentThread() to target the running 'Thread-1'.
           |     - It opens Thread-1's private 'threadLocals' map instance variable.

           |     - KEY: 'SpringTransactionKey' (The static DataSource resource identifier).
           |     - VALUE: 'Connection #1' (Holding the open OS TCP database socket descriptor).
           |     The thread's memory backpack is now officially loaded!
           v (12) Proxy forwards the CPU Program Counter register back into the raw object's methodB() address.
 [ RAW OrderService BEAN ] ----------------------------------------------------------------------+

           |                                                                                     |
           +---> (13) Thread-1 pushes a new Stack Frame for methodB() onto its private Memory Stack.

           |     Repository query executes. Hibernate needs to stream raw SQL statements.
           v
 [ HIBERNATE ENGINE ]
           |
           v (14) Invokes Spring's Core Infrastructure to search for an active database socket.
 [ TransactionSynchronizationManager ]

           |
           v (15) Triggers 'Thread.currentThread().threadLocals.get(SpringTransactionKey)'.
           |      - JVM checks the physical CPU register to confirm Thread-1 is running.
           |      - It reaches directly into Thread-1's private 'threadLocals' map to inspect keys.
           v
 [ Thread-1 STORAGE BACKPACK ] 

           |
           +---> (16) MAP LOOKUP MECHANICS:
           |     The map successfully finds 'SpringTransactionKey' mapped to 'Connection #1'.
           v
 [ HIBERNATE ENGINE ] <--- (17) RESULT: Successfully extracts Connection #1 from the backpack!

           |
           v (18) Hibernate reuses Connection #1. The JDBC driver opens the embedded OS TCP socket
           |      descriptor and streams query bytes into an active, uncommitted transaction context.
           v
  [ MASTER CGLIB PROXY ] <--- (19) Method finishes. Execution flows back out to the proxy tier on Thread-1.
           |
           +---> (20) Proxy invokes 'connection1.commit()', pushing the commit packet over the TCP wire.
           +---> (21) TEARDOWN: Proxy calls 'unbindResource()', completely clearing Connection #1 out
                 of Thread-1's private 'threadLocals' map and returns the connection to HikariCP.



package com.architect.masterclass;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    // Target 1: Bypassed by the proxy for its local annotation, but carries context
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void methodB() {
        System.out.println("Executing database operations inside methodB...");
    }

    // Target 2: Outer method initializes the transaction on the thread
    @Transactional
    public void methodC() {
        System.out.println("Outer transaction initialized successfully for methodC...");
        
        // This is implicitly processed by the JVM as: this.methodB();
        methodB(); 
    }
}

 [ TOMCAT THREAD POOL ]


           |
           v (1) Allocates Native Worker Thread 'Thread-1'. 
           |     JVM assigns a private Memory Stack and initializes an empty 'threadLocals' map
           |     (the "Memory Backpack") inside the Thread-1 object structure [1, 2, 3].
      [ Thread-1 ]

           |
           v (2) Executes external call: 'orderService.methodC()' targeting the proxy bean wrapper.
  [ SPRING CGLIB PROXY ] ------------------------------------------------------------------------+
           |                                                                                     |
           +---> (3) Scans internal configuration metadata registry for 'methodC()'.             |

           |     RESULT: @Transactional annotation detected! Interception Engine activates.      |
           +---> (4) The proxy pauses execution, requests an active connection ('Connection #1') |

           |     from the HikariCP pool, and sets 'connection1.setAutoCommit(false)' over the TCP wire.
           +---> (5) CORE BINDING ACT: The proxy invokes 'TransactionSynchronizationManager.bindResource()'.
           |     - JVM uses Thread.currentThread() to target the running 'Thread-1'.
           |     - It opens Thread-1's private 'threadLocals' map instance variable.

           |     - KEY: 'SpringTransactionKey' (The static DataSource resource identifier).
           |     - VALUE: 'Connection #1' (Holding the open OS TCP database socket descriptor).
           |     The thread's memory backpack is now officially loaded!                          |
           v (6) Proxy forwards the CPU Program Counter (PC) register straight to the raw methodC() address.
 [ RAW OrderService BEAN ] ----------------------------------------------------------------------+

           |                                                                                     |
           +---> (7) Thread-1 pushes a new Stack Frame onto its private Memory Stack to run methodC().

           |     It executes plain, outer transactional business logic.                         |
           |                                                                                     |
           +---> (8) Encounters 'methodB()' line. JVM executes 'invokevirtual' bytecode instruction.

           |     CRITICAL POINT: This uses the implicit Java 'this' reference.                  |
           |                                                                                     |
           v (9) The CPU Program Counter executes a local memory jump straight to methodB().    |
  [ METHOD B BODY ] <----------------------------------------------------------------------------+

           |
           v (10) Thread-1 pushes a new Stack Frame for methodB() onto its private Memory Stack.
           |      The customized 'Propagation.REQUIRES_NEW' on methodB() is COMPLETELY IGNORED.
           |      Repository query executes. Hibernate needs to stream raw SQL statements.
 [ HIBERNATE ENGINE ]

           |
           v (11) Invokes Spring's Core Infrastructure to search for an active database socket.
 [ TransactionSynchronizationManager ]
           |
           v (12) Triggers 'Thread.currentThread().threadLocals.get(SpringTransactionKey)'.

           |      - JVM checks the physical CPU register to confirm Thread-1 is running.
           |      - It reaches directly into Thread-1's private 'threadLocals' map to inspect keys.
           v
 [ Thread-1 STORAGE BACKPACK ] 

           |
           +---> (13) MAP LOOKUP MECHANICS:
           |     Even though the proxy was bypassed for methodB, Thread-1 is still carrying 
           |     Connection #1 inside its backpack from Step 5. The lookup finds Connection #1.
           v
 [ HIBERNATE ENGINE ] <--- (14) RESULT: Successfully extracts Connection #1 from the backpack!

           |
           v (15) Hibernate reuses Connection #1. The JDBC driver opens the embedded OS TCP socket
           |      descriptor and streams query bytes safely within the transaction context of methodC().
           v
  [ MASTER CGLIB PROXY ] <--- (16) Execution finishes methodB(), returns to methodC(), and flows 

           |                      back out to the outermost proxy tier on Thread-1.
           +---> (17) Proxy invokes a single unified 'connection1.commit()', pushing the commit packet
           |     over the TCP wire to finalize changes from BOTH methods at the exact same time.
           +---> (18) TEARDOWN: Proxy calls 'unbindResource()', completely clearing Connection #1 out
                 of Thread-1's private 'threadLocals' map and returns the connection to HikariCP.




