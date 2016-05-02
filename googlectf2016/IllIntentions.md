# Ill Intentions

*Solved by @\_pox\_ and @cogitoergor00t*

We have a file called **illintentions.apk**

Let's decompile it

```
apktool d illintenions.apk
```

Ok, so now we have the smali code and some useful informations in *AndroidManifest.xml*

Let's have a look

```
cat AndroidManifest.xml
```

Interesting, we have 4 Activities in this application:

```
<activity android:label="Main Activity" android:name="com.example.application.MainActivity">
<activity android:label="Activity: Is This The Real One" android:name="com.example.application.IsThisTheRealOne"/>
<activity android:label="This Is The Real One" android:name="com.example.application.ThisIsTheRealOne"/>
<activity android:label="Definitely Not This One" android:name="com.example.application.DefinitelyNotThisOne"/>
```

Ok, time to use the emulator:

```
adb install illintentions.apk
```

and start the application.
We can see only the first activity **MainActivity** and nothing more: one interesting thing we can do is to call the other activities using 

```
adb shell am start -n com.example.hellojni/com.example.application.IsThisTheRealOne
adb shell am start -n com.example.hellojni/com.example.application.ThisIsTheRealOne
adb shell am start -n com.example.hellojni/com.example.application.DefinitelyNotThisOne
```

(We have to add *android:exported="true"* in the activity-tag or using a device with root permissions)

The only thing we can do when that activity is changed, is to click the button "Broadcast Intent".

Time to read some code... we can see the application is using JNI, so maybe the interesting functions are in those native libraries. We don't want to spend time reversing that code, so let's read some **smali**. Ok, in all the activities there is a onClick handler, let's have a look:

```
    .method public onClick(Landroid/view/View;)V
    ...

    const v6, 0x7f030006
    invoke-virtual {v5, v6}, Landroid/content/res/Resources;->getString(I)Ljava/lang/String;
    move-result-object v5
    invoke-virtual {v4, v5}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;
    move-result-object v4
    const-string v5, "YSmks"
    invoke-virtual {v4, v5}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    ... 

    invoke-static {v4}, Lcom/example/application/Utilities;->doBoth(Ljava/lang/String;)Ljava/lang/String;
    move-result-object v1

    ...
    # Native function
    invoke-virtual {v5, v0, v1, v2}, Lcom/example/application/ThisIsTheRealOne;->orThat(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
    move-result-object v5

```

Ok, we can see that the activity loads some strings, it appends "YSmks", calls doBoth function in the Utilities class and so on... The interesting part is the native function call.
If we click the button in the activity screen, we can't see any output: let's add some smali code to see what is the results of the native function calls in the three activities.

We can see in the last part of smali, the result of **ThisIsTheRealOne;->orThat** is moved in **v5** register, so we can simply print the value of that register.

```
    const-string v0, "BackInBlack - "
    invoke-static {v0, v5}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I
    move-result v0
```

We can add these 3 lines of code under the **move-result-object v5** instruction, and repeat it in the other activities (we have to find the right native function call and then add the correct code)

The **native functions** we want to hook are:

*DefinitelyNotThisOne$1.smali*
```
    invoke-virtual {v5, v1, v2}, Lcom/example/application/DefinitelyNotThisOne;->definitelyNotThis(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
    move-result-object v5
```

*IsThisTheRealOne$1.smali*
```
    invoke-virtual {v6, v0, v1, v2}, Lcom/example/application/IsThisTheRealOne;->perhapsThis(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
    move-result-object v6
```

and *ThisIsTheRealOne$1.smali*
```
    invoke-virtual {v5, v0, v1, v2}, Lcom/example/application/ThisIsTheRealOne;->orThat(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
    move-result-object v5
```
Ok, time to see if we have done the things correctly.

```
# Build the apk
apktool b illintentions/

cp illintentions/dist/illintentions.apk illintentions-unsigned-unalligned.apk

# Create a key
keytool -genkey -v -keystore BIB.keystore -alias backinblack -keyalg RSA -keysize 2048 -validity 10000

# Sign
jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore BIB.keystore illintentions-unsigned-unalligned.apk backinblack

# Verify
jarsigner -verify -verbose -certs illintentions-unsigned-unalligned.apk

# Align
zipalign -v 4 illintentions-unsigned-unalligned.apk final.apk

# Install
adb install -r final.apk
```

Ok, now the point is:

* Start the application
* Using adb to call the activities
* Click the button
* Look with logcat if there is something interesting

```
adb shell am start -n com.example.hellojni/com.example.application.ThisIsTheRealOne

In the log we can see

05-02 03:57:04.680  2245  2245 D BackInBlack - : KeepTryingThisIsNotTheActivityYouAreLookingForButHereHaveSomeInternetPoints!
```

Ok, try harder

```
adb shell am start -n com.example.hellojni/com.example.application.DefinitelyNotThisOne

Again,

05-02 04:02:54.276  2323  2323 D BackInBlack - : Told you so!
```

Last try...

```
adb shell am start -n com.example.hellojni/com.example.application.IsThisTheRealOne

And finally,

05-02 04:04:26.301  2390  2390 D BackInBlack - : Congratulation!YouFoundTheRightActivityHereYouGo-CTF{IDontHaveABadjokeSorry}
```

**Flag** : **CTF{IDontHaveABadjokeSorry}**
