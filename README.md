# Open Rail Data Stomp client for Java

## An example Stomp client for Java

**NB** The popular Gorizza library does not support receipt of binary data over STOMP, so will not be able to consume the GZIP stream coming across this wire.

Make use of the https://github.com/fusesource/stompjms library to add STOMP support to the lower level http://activemq.apache.org library.

##### build.gradle
```
compile (group: 'org.apache.activemq', name: 'activemq-client', version: '5.15.0')
compile group: 'org.fusesource.stompjms', name: 'stompjms-client', version: '1.19'
```

##### src/main/java/darwinStomp/StompClient
``` java
package darwinStomp;

import java.util.logging.Level;
import java.util.logging.Logger;
import javax.jms.Connection;
import javax.jms.JMSException;
import javax.jms.MessageConsumer;
import javax.jms.Queue;
import javax.jms.Session;
import org.fusesource.stomp.jms.StompJmsConnectionFactory;

public class StompClient implements Runnable {

    private static final Logger LOG = Logger.getLogger(StompClient.class.getName());

    private final String BROKER_URI = "tcp://datafeeds.nationalrail.co.uk:61613";
    private final String QUEUE_NAME = "Your security key from My Feeds";

    public static void main(String[] args) {
        new StompClient().run();
    }

    @Override
    public void run() {

        StompJmsConnectionFactory connectionFactory = new StompJmsConnectionFactory();

        connectionFactory.setBrokerURI(this.BROKER_URI);

        Connection connection = null;
        Session session = null;
        MessageConsumer consumer = null;

        try {
            connection = connectionFactory.createConnection("d3user", "d3password");

            session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
            Queue queue = session.createQueue(this.QUEUE_NAME);
            consumer = session.createConsumer(queue);

            System.out.println("Connected to STOMP " + this.BROKER_URI);

            consumer.setMessageListener(new MessageHandler());

            while (!Thread.interrupted()) {
            }

        } catch (JMSException ex) {
            Logger.getLogger(StompClient.class.getName()).log(Level.SEVERE, null, ex);
            System.out.println("Thread interupted by exception or shutdown");
        } finally {
            if (consumer != null) {
                try {
                    consumer.close();
                } catch (JMSException ex) {
                    LOG.log(Level.SEVERE, null, ex);
                }
            }
            if (session != null) {
                try {
                    session.close();
                } catch (JMSException ex) {
                    LOG.log(Level.SEVERE, null, ex);
                }
            }
            if (connection != null) {
                try {
                    connection.close();
                } catch (JMSException ex) {
                    LOG.log(Level.SEVERE, null, ex);
                }
            }
        }
    }
}

```

##### src/main/java/darwinStomp/MessageHandler
``` java
package darwinStomp;

import javax.jms.BytesMessage;
import javax.jms.JMSException;
import javax.jms.Message;
import javax.jms.MessageListener;
import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.Reader;
import java.util.zip.GZIPInputStream;

public class MessageHandler implements MessageListener {
    @Override
    public void onMessage(Message message) {
        String xmlString = convertToXmlString((BytesMessage) message);
        System.out.println(xmlString);
    }

    private String convertToXmlString(BytesMessage bytesMessage) {
        if (bytesMessage != null) try {
            long l = bytesMessage.getBodyLength();
            byte bytesArray[] = new byte[(int) l];
            bytesMessage.readBytes(bytesArray);
            Reader streamReader = null;
            try {
                streamReader = new InputStreamReader(new GZIPInputStream(new ByteArrayInputStream(bytesArray)));
                StringBuilder stringBuilder = new StringBuilder();
                char cb[] = new char[1024];
                int s = streamReader.read(cb);
                while (s > -1) {
                    stringBuilder.append(cb, 0, s);
                    s = streamReader.read(cb);
                }
                return stringBuilder.toString();
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                if (streamReader != null) {
                    streamReader.close();
                }
            }
        } catch (IOException ex) {
            System.out.println("Failed to parse message");
            ex.printStackTrace();
        } catch (JMSException ex) {
            System.out.println("Failed to parse message");
            ex.printStackTrace();
        }

        return null;
    }
}
```
