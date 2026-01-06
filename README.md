Please note that it is illegal to hire minors, as it is a violation of child labor laws especially in the State of Texas

This example app tracks labor based on the work reqester or employer, a data source and a worker. It is designed to allow metadata processing systems and public ledgers like blockchains, to be the primary means of tracking work and validating work.

Specifically the app expects to recieve information from the XRP, BTC, ETH, ONT, and ONG markets as well as other big data providers.

It allows employers to register and post work requests. 

The app allows workers to register and to accept work requests. 

It then allows workers to close work requests. 

The app then query's a remote server which is in turn tasked with verifying the work using metadata that it has access to. For example, when the worker accepts the work request, the app can send a signal to the remote server. The remote server can then begin purchasing data from the Ontology network or another ETH based network.

In the above example it is assumed that a "loading network" is monitoring the worker and generating metadata about the behaviors the worker is performing. The loading network is further assumed to populate the Ontology or other ETH based network with the data. Upon loading the data onto the Ontology network or ETH network a transaction is recorded in the blockcahin. Thi transaction should be populated by the loading network to include the unique identifier assigned to the worker in the app. 

When the user closes the work request the app can send a signal to the remote server. The remote server can then query the Ontology or other ETH based network for the unique identifier of the worker in order to validate that the data set generated through the task was populated or loaded onto the Ontology or other ETH network. 

The remote server can then employ some manner of data analysis to verify that the transaction's representative of the worker's behavior actually indicate that the worker completed the requested tasks. This is largely an AI task probably best suited for a cloud system as opposed to a task that should be performed on a hand held device. 

Once the remote server has completed verifying that the work was completed it can send a signal back to the app so that the employer recieves a notification that the work is completed. 

The employer can then use the app to post payment. At this point the app and the remote server will act as a payment gateway (authorize.net for example) which take the alloted sum from the employer's specified account. The app and remote server will store the fiscal dollars in a merchant account. The merchant account will then be instructed by the app to transfer the funds to the account specified by the worker. 

This last part is highly configurable and and could for example be implemented with Stripe, Paypal, CashApp, Venmo, or other API's. 

-------------------------------------------------------------------------------------------------------------------------------------------------
 
The following is another brief business analyst perspective of the app: 
------------------------------------------------------------------

The app needs to wait for a remote server to assign a given worker a set number of hours of work (1 hour by default). The app needs to receive a basic identifier unique to the employer from a remote server. When the work request is received the app needs to assign it to a worker. The worker must accept the work request. The worker must then perform the task the app has assigned it in the real world while a series of remote systems collect their metadata. The app then needs to allow the worker to tell it when the job is completed. When the worker tells it the job is completed the remote server will request payment from the work recipient or employer. The worker can then tell the app where to send their payment. The app will tell the employer where to send the payment. The app then allows the employer to post payment to the worker's specified account. The app then posts a message notifying the worker that their payment has been posted.




