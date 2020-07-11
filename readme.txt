This fixed EJBTHREE-1177

https://jira.jboss.org/jira/browse/EJBTHREE-1177

Wrong "in method flag" on stack after first invocation of a jca endpoint

17:24:11,609 ERROR [AllowedOperationsAssociation] getCallerPrincipal should not be access from this bean method: IN_EJB_CREATE, allowed is [IN_EJB_TIMEOUT, IN_BUSINESS_METHOD, IN_SERVICE_ENDPOINT_METHOD]
java.lang.IllegalStateException: getCallerPrincipal should not be access from this bean method: IN_EJB_CREATE
        at org.jboss.ejb.AllowedOperationsAssociation.assertAllowedIn(AllowedOperationsAssociation.java:158)
        at org.jboss.ejb.StatelessSessionEnterpriseContext$SessionContextImpl.getCallerPrincipal(StatelessSessionEnterpriseContext.java:221)
but the invocation currently is not in an call to ejbCreate().

This is caused by MessageDrivenEnterpriseContext where IN_EJB_CREATE is pushed on the stack which is never removed and so remains as top level in method flag.

Code of org.jboss.ejb.MessageDrivenEnterpriseContext, revision 57209

(line 70) public MessageDrivenEnterpriseContext(Object instance, Container con)
      throws Exception
   {
      super(instance, con);
      
      ctx = new MessageDrivenContextImpl();
      ((MessageDrivenBean)instance).setMessageDrivenContext(ctx);

      try
      {
(line 80) AllowedOperationsAssociation.pushInMethodFlag(IN_EJB_CREATE);
         Method ejbCreate = instance.getClass().getMethod("ejbCreate", new Class[0]);
         ejbCreate.invoke(instance, new Object[0]);
      }
      catch (InvocationTargetException e)
      {
         Throwable t = e.getTargetException();
         
         if (t instanceof RuntimeException) {
            if (t instanceof EJBException) {
               throw (EJBException)t;
            }
            else {
               // Transform runtime exception into what a bean *should* have thrown
               throw new EJBException((RuntimeException)t);
            }
         }
         else if (t instanceof Exception) {
            throw (Exception)t;
         }
         else if (t instanceof Error) {
            throw (Error)t;
         }
         else {
            throw new org.jboss.util.NestedError("Unexpected Throwable", t);
         }
      }
   }

On first delivery the MDB pool is created and the constructor of org.jboss.ejb.MessageDrivenEnterpriseContext is invoked, where IN_EJB_CREATE is pushed in line 80. As far as I can see, the flag is not popped from the stack.

Possibly the complementary AllowedOperationsAssociation.popInMethodFlag() in a finally block after 106 is missing.

(I checked the latest SVN revision (66439 ) of MessageDrivenEnterpriseContext and have not seen any changes in this area.)

Regards
Conni
