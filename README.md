# JMeter-Load-Test-Jenkins

This document illustrate the steps to create Load Test Plan in JMeter, as well as integration in Jenkins to run the test via shell script.

*Add Test Plan*
```
JMeter->Test Plan->Add->Config Element->User Defined Variables
server_ip=${__P(server_ip,10.0.0.123)}
server_port=${__P(server_port,1234)}
api_create_user=CreateUser
test_amount=10
pause_test_for_1_second=1000

JMeter->Test Plan->Add->Listener->View Result Tree

JMeter->Test Plan->Add->Threads(Users)->Thread Group, name it "Load Test"
JMeter->Test Plan->Load Test->Add->Logic Controller->Simple Controller
```

*Add BeanShell PreProcessor*
```
int threadNumber=ctx.getThreadNum();//get current thread number
int new_user_id = ${user_id_start}+threadNumber;//increment user id by 1
vars.put("user_id",String.valueOf(new_user_id));//store new user id
```

*Add BeanShell Sampler*
Put java code to execute the logic, e.g. open socket

```
import java.net.Socket;
import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.nio.file.CopyOption;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.nio.file.StandardCopyOption;
import java.io.DataOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;

int user_id = Integer.valueOf(vars.get("user_id"));
StringBuffer command_template = new StringBuffer("{\"command\":\"registerUser\",\"user_id\":123}");
StringBuffer command = new StringBuffer(command_template.toString().replace("\"user_id\": 123,","\"user_id\": "+ user_id));

String IP = String.valueOf("${server_ip}");
int PORT = Integer.valueOf(${server_port});

try {
	Socket client = new Socket(IP,PORT);

	InputStream in = client.getInputStream();
	OutputStream out = client.getOutputStream();
	DataOutputStream writer = new DataOutputStream(out);
	ByteBuffer byteBuffer = ByteBuffer.allocate(4);
	byteBuffer.order(ByteOrder.LITTLE_ENDIAN);
	writer.write(byteBuffer.putInt(command.toString().length()).array());
	writer.writeBytes(command.toString());
	writer.flush();

	int attemps = 0;
	while (in.available()==0 && attemps<8){
		attemps++;
		Thread.sleep(1);
	}

	byteBuffer.clear();
	in.close();
	out.close();
	client.close();
} catch (IOException e) {
} catch (InterruptedException ex) {
}

```

*Add Logic Controller*
Logic Controller->Loop Controller, Config Element->Counter, Sampler->Test Action, Logic Controller->Once Only Controller
set Loop Controller's Count to ${test_amount}
set Counter's Start=1, Increment=1, Maximum=${test_amount}, Reference Name=test_counter_current which can be referenced later
set Test Action's Name=Delay, Action=Pause, Duration=${pause_test_for_1_second}
set to put a Debug Sampler inside Once Only Controller to sample the result once

*Add Module Controller to reference to an existing controller for reuse purpose*

Add Sampler->HTTP Request, Post Processors->JSON Path Extractor
set JSONPath Expression=$.results[1].return.result
set JSONPath Expression=$.results[4].return.resultss[?(@.id==101)].identifier

*In Jenkins, set the following as String Parameter*
SERVER=dev@10.0.0.123@/path/to/responder
LOG=Load_Test_Log_${BUILD_NUMBER}.log
JTL=Load_Test_Result_${BUILD_NUMBER}.jtl

*Execute Shell*
SERVER_IP=$(echo ${SERVER} | cut -d- -f2 | cut -d@ -f2)
SERVER_PATH=$(echo ${SERVER} | cut -d- -f3 | cut -d@ -f3)
JVM_ARGS="-Xms${MEMORY_MIN} -Xmx${MEMORY_MAX}" /usr/lib/${JMETER} -n -t /var/lib/Load-Test/${JMX} -l ${JTL} -j ${LOG} -Jserver_ip=$SERVER_IP -Jserver_path=$SERVER_PATH -Juser_id_start=$USER_ID_START -Jramp_up_time=${RAMP_UP_TIME} 
