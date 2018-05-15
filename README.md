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

import org.fusesource.stomp.jms.StompJmsConnectionFactory;
import javax.jms.*;

public class StompClient implements Runnable {

    public static void main(String[] args) throws Exception {
        new StompClient().run();
    }

    @Override
    public void run() {
        String brokerUri = "tcp://datafeeds.nationalrail.co.uk:61613";
        String QUEUE_NAME = "Your security key from My Feeds"

        StompJmsConnectionFactory connectionFactory = new StompJmsConnectionFactory();
        connectionFactory.setBrokerURI(brokerUri);

        Connection connection = null;
        Session session = null;
        MessageConsumer consumer = null;

        try {
            connection = connectionFactory.createConnection("d3user", "d3password");
            connection.start();

            session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
            Queue queue = session.createQueue(QUEUE_NAME);
            consumer = session.createConsumer(queue);

            System.out.println("Connected to STOMP " + brokerUri);

            consumer.setMessageListener(new MessageHandler());

            while (!Thread.interrupted()) {}

            try {
                if (consumer != null) {
                    consumer.close();
                }

                if (session != null) {
                    session.close();
                }

                if (connection != null) {
                    connection.close();
                    connection = null;
                }
            } catch (JMSException ex) {
                System.out.println("Got exception on shutdown");
                ex.printStackTrace();
            }

            System.out.println("Thread was interrupted!");


        } catch (JMSException e) {
            e.printStackTrace();
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

    private static final Logger LOG = Logger.getLogger(MessageHandler.class.getName());

    @Override
    public void onMessage(Message message) {
        String xmlString = convertToXmlString((BytesMessage) message);
        System.out.println(xmlString);
    }

    private String convertToXmlString(BytesMessage bytesMessage) {
        if (bytesMessage != null) try {
            long length = bytesMessage.getBodyLength();
            byte[] bytesArray = new byte[(int) length];
            bytesMessage.readBytes(bytesArray);
            Reader streamReader = null;
            try {
                streamReader = new InputStreamReader(new GZIPInputStream(new ByteArrayInputStream(bytesArray)));
                StringBuilder stringBuilder = new StringBuilder();
                char[] charBuffer = new char[1024];
                int size = streamReader.read(charBuffer);
                while (size > -1) {
                    stringBuilder.append(cb, 0, size);
                    size = streamReader.read(charBuffer);
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
            LOG.log(Level.SEVERE, null, ex);
        } catch (JMSException ex) {
            LOG.log(Level.SEVERE, null, ex);
        }

        return null;
    }
}
```
