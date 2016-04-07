---
title: Preprocessing and Postprocessing IRPs
description: Preprocessing and Postprocessing IRPs
ms.assetid: a0e14ae6-a06e-4c24-8b64-b56f485cf9ff
keywords: ["preprocessing IRPs WDK KMDF", "postprocessing IRPs WDK KMDF", "WDM IRPs WDK KMDF , preprocessing and postprocessing", "IRPs WDK KMDF , preprocessing and postprocessing"]
---

# Preprocessing and Postprocessing IRPs


\[Applies to KMDF only\]

If your driver must intercept an I/O request packet (IRP) before or after the framework handles the IRP, the driver can call [**WdfDeviceInitAssignWdmIrpPreprocessCallback**](https://msdn.microsoft.com/library/windows/hardware/ff546043) to register an [*EvtDeviceWdmIrpPreprocess*](https://msdn.microsoft.com/library/windows/hardware/ff540925) event callback function for a major I/O function code and, optionally, for specific minor I/O function codes that are associated with the major code. Subsequently, the framework calls the driver's *EvtDeviceWdmIrpPreprocess* callback function whenever the driver receives an IRP that contains a specified major and minor function code.

The [*EvtDeviceWdmIrpPreprocess*](https://msdn.microsoft.com/library/windows/hardware/ff540925) callback function can do whatever is necessary to preprocess the IRP, and then it must call [**WdfDeviceWdmDispatchPreprocessedIrp**](https://msdn.microsoft.com/library/windows/hardware/ff546927) to return the IRP to the framework unless the driver is [handling an IRP that the framework does not support](handling-an-irp-that-the-framework-does-not-support.md).

After the driver calls [**WdfDeviceWdmDispatchPreprocessedIrp**](https://msdn.microsoft.com/library/windows/hardware/ff546927), the framework processes the IRP in the same way that it would have if the driver had not provided an [*EvtDeviceWdmIrpPreprocess*](https://msdn.microsoft.com/library/windows/hardware/ff540925) callback function. If the IRP's I/O function code is one that the framework passes to drivers, the driver will receive the IRP again as a request object.

If the driver needs to postprocess the IRP after a lower-level driver completes the IRP, the driver's [*EvtDeviceWdmIrpPreprocess*](https://msdn.microsoft.com/library/windows/hardware/ff540925) callback function can call [**IoSetCompletionRoutine**](https://msdn.microsoft.com/library/windows/hardware/ff549679) to set an [*IoCompletion*](https://msdn.microsoft.com/library/windows/hardware/ff548354) routine before it calls [**WdfDeviceWdmDispatchPreprocessedIrp**](https://msdn.microsoft.com/library/windows/hardware/ff546927).

After your driver calls [**WdfDeviceInitAssignWdmIrpPreprocessCallback**](https://msdn.microsoft.com/library/windows/hardware/ff546043), the framework causes the I/O manager to add an additional [I/O stack location](https://msdn.microsoft.com/library/windows/hardware/ff551821) to all IRPs so that the [*EvtDeviceWdmIrpPreprocess*](https://msdn.microsoft.com/library/windows/hardware/ff540925) callback function can set an [*IoCompletion*](https://msdn.microsoft.com/library/windows/hardware/ff548354) routine. The callback function must update the IRP's I/O stack location pointer before it calls [**WdfDeviceWdmDispatchPreprocessedIrp**](https://msdn.microsoft.com/library/windows/hardware/ff546927).

### Calling WdfDeviceWdmDispatchPreprocessedIrp

Because the I/O manager adds an additional I/O stack location to the IRP, the [*EvtDeviceWdmIrpPreprocess*](https://msdn.microsoft.com/library/windows/hardware/ff540925) callback function must call [**IoSkipCurrentIrpStackLocation**](https://msdn.microsoft.com/library/windows/hardware/ff550355) or [**IoCopyCurrentIrpStackLocationToNext**](https://msdn.microsoft.com/library/windows/hardware/ff548387) (to set up the next I/O stack location in the IRP) before calling [**WdfDeviceWdmDispatchPreprocessedIrp**](https://msdn.microsoft.com/library/windows/hardware/ff546927).

If your driver is preprocessing an IRP, but not postprocessing the IRP, the driver does not need to set an [*IoCompletion*](https://msdn.microsoft.com/library/windows/hardware/ff548354) routine for the IRP and can call [**IoSkipCurrentIrpStackLocation**](https://msdn.microsoft.com/library/windows/hardware/ff550355), as the following code example shows.

```
NTSTATUS
  EvtDeviceMyIrpPreprocess(
    IN WDFDEVICE Device,
    IN OUT PIRP Irp
    )
{
//
// Perform IRP preprocessing operations here.
//
...
//
// Deliver the IRP back to the framework. 
//
IoSkipCurrentIrpStackLocation(Irp);
return WdfDeviceWdmDispatchPreprocessedIrp(Device, Irp);
}
```

If your driver is postprocessing the IRP, the driver must call [**IoCopyCurrentIrpStackLocationToNext**](https://msdn.microsoft.com/library/windows/hardware/ff548387), and then it must call [**IoSetCompletionRoutine**](https://msdn.microsoft.com/library/windows/hardware/ff549679) to set an [*IoCompletion*](https://msdn.microsoft.com/library/windows/hardware/ff548354) routine for the IRP, as the following code example shows.

```
NTSTATUS
  EvtDeviceMyIrpPreprocess(
    IN WDFDEVICE Device,
    IN OUT PIRP Irp
    )
{
//
// Perform IRP preprocessing operations here, if needed.
//
...
//
// Set a completion routine and deliver the IRP back to
// the framework. 
//
IoCopyCurrentIrpStackLocationToNext(Irp);
IoSetCompletionRoutine(
                       Irp,
                       MyIrpCompletionRoutine,
                       NULL,
                       TRUE,
                       TRUE,
                       TRUE
                      );
return WdfDeviceWdmDispatchPreprocessedIrp(Device, Irp);
}
```

Your driver must not call [**IoCopyCurrentIrpStackLocationToNext**](https://msdn.microsoft.com/library/windows/hardware/ff548387) (and therefore must not set an [*IoCompletion*](https://msdn.microsoft.com/library/windows/hardware/ff548354) routine) if the device object handle that the driver's [*EvtDeviceWdmIrpPreprocess*](https://msdn.microsoft.com/library/windows/hardware/ff540925) callback function receives represents a physical device object (PDO), and if the IRP's major function code is IRP\_MJ\_PNP or IRP\_MJ\_POWER. Otherwise, [Driver Verifier](https://msdn.microsoft.com/library/windows/hardware/ff545448) will report an error.

For more information about when to call [**IoCopyCurrentIrpStackLocationToNext**](https://msdn.microsoft.com/library/windows/hardware/ff548387), [**IoSkipCurrentIrpStackLocation**](https://msdn.microsoft.com/library/windows/hardware/ff550355), and [**IoSetCompletionRoutine**](https://msdn.microsoft.com/library/windows/hardware/ff549679), see [Passing IRPs down the Driver Stack](https://msdn.microsoft.com/library/windows/hardware/ff558781).

 

 

[Send comments about this topic to Microsoft](mailto:wsddocfb@microsoft.com?subject=Documentation%20feedback%20%5Bwdf\wdf%5D:%20Preprocessing%20and%20Postprocessing%20IRPs%20%20RELEASE:%20%284/5/2016%29&body=%0A%0APRIVACY%20STATEMENT%0A%0AWe%20use%20your%20feedback%20to%20improve%20the%20documentation.%20We%20don't%20use%20your%20email%20address%20for%20any%20other%20purpose,%20and%20we'll%20remove%20your%20email%20address%20from%20our%20system%20after%20the%20issue%20that%20you're%20reporting%20is%20fixed.%20While%20we're%20working%20to%20fix%20this%20issue,%20we%20might%20send%20you%20an%20email%20message%20to%20ask%20for%20more%20info.%20Later,%20we%20might%20also%20send%20you%20an%20email%20message%20to%20let%20you%20know%20that%20we've%20addressed%20your%20feedback.%0A%0AFor%20more%20info%20about%20Microsoft's%20privacy%20policy,%20see%20http://privacy.microsoft.com/default.aspx. "Send comments about this topic to Microsoft")



