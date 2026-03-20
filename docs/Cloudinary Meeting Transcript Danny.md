United Talent Agency / Cloudinary

undefined undefined
Recorded on Mar 19, 2026 via , 33m



Participants

Cloudinary
Hannah Baldovino, Account Manager
Danny Bond, Senior Digital CSM

Metrolinx
Ahmed Salam, Director, Delivery Management

Other
Edgar Alvarez
Francisco Perez
Leo Breckenridge



Transcript

0:00 | Hannah
hi. How are you? 

0:04 | Edgar
Hey, how are you doing? 

0:07 | Hannah
I'm doing well. I got to apologize. I'm sorry, it's a seven 30 a M here for me. And so this is kind of an hour where I'm sort of rushing my kids out the door and as you could see, so, I have my camera off, but I did just want to quickly introduce myself when the team joined… and honestly, it'll really just be you guys chatting with Danny directly. So, but yeah, I just, but just wanted to say hi. 

0:38 | Edgar
Hi, thank you. Thanks for joining. 

0:42 | Danny
Hey. 

0:43 | Edgar
Danny. Nice to meet you. 

0:45 | Danny
You too. Hey, Helen. 

0:47 | Hannah
Hi, Danny. 

0:52 | Edgar
All right. We have Leo Ahmed and probably Francisco will join. Let me make a very quick introduction because we are really short on time. I'm Edgar Alvarez, I'm located in Medellin Colombia. I'm a solution, senior solution architect for epan, Leo here is our senior business architect, senior business analyst. There you go, Leo, and we have Ahmed, which is our project manager for this specific interaction with Uta. If Francisco joins, I hope he's our devops Guy. All right, just in case. All right, Danny. Yeah, briefly tell us your role. 

1:37 | Danny
Yeah. So I'm Danny bond, one of cloudos, technical, customer success managers. I'm based in the UK as you can probably tell from my accent. I've been with the company four years in the role of well, in this particular role, it's really my responsibility mostly to help our customers align their business objectives with our capabilities, features, functionality, etc, provide advice on use cases and really just help them extract the maximum possible from the platform. So I'm aware that today is probably going to be a fairly technical conversation. I have had a look through some of the questions which I believe you've been working with mo in support. Until now, it's not clear to me actually what answers to what questions you've actually received. So I have gone through and compiled what I can. I think in reality the goal today will be for us to go through the questions that you have for me to answer what I can to collate anything else and then quite likely formally request a professional service engagement so that we can actually map this out in more detail. But I think I have most of the questions covered and I can share the actual documentation and the answers to the questions after the call as well. 

2:49 | Edgar
Awesome. Cool. Thank you so much. Yeah, just to give you an overall context. The main goal is to bring your own bucket. So Uta will have the S tree bucket and we need to migrate the files. 

3:03 | Danny
Right. Yep. And. 

3:04 | Edgar
Keep everything working in production. Yep, where else? Unfortunately, most of the cases, Uta wraps the Cloudinary URL with their apm, so. 

3:19 | Danny
There. 

3:19 | Edgar
shouldn't be any disruptions because it is masked. So in that order of ideas, whatever we move on behind the scenes, it will not affect production, right? 

3:30 | Danny
Right. 

3:31 | Edgar
That's one thing we have to keep in mind now, in… this byob context… we understand we need a new environment right from Cloudinary. So the new environment will be or let's say testing environment. We do have, Uta has a testing environment for their applications. They have around seven applications. So we would like to have a new environment in let's say testing, right? So we can start moving and making some pocs and make sure everything works. Yeah. Now, can you elaborate a little bit more about this new environment? Does that mean, does it mean imagine production, right? We have three terabytes of assets, all the metadata, all they have their URL, public ids, all this stuff. So we create a new environment. What does that mean? Does it mean it copies the metadata? It means it copies the settings it means or it's an empty thing? What happens when you create a? 

4:42 | Danny
New environment? Yeah. So when you create a new environment, you have the option to create a new environment based on the configuration of an existing environment. However, the big difference here is to bring your own bucket. That is something which would need to be configured by us specifically through professional services, so we can create the environment. But then it's going to need to be configured to be able to use the customer's own S3 bucket beyond that. And then in terms when we start looking at the migration, there are different approaches. I believe the one that's been mostly discussed with support and probably makes sense. But I would probably lean on professional services for additional advice here would be to use our cli tool. So there are various functions within that tool which actually allow you, for example, the clone function will allow you to directly copy the assets, not just the binaries, but also the associated data related to those assets between environments. So I think that's probably going to be the sensible approach here. Obviously pending further discovery and making sure that we're not missing anything. 

5:56 | Edgar
So in order of steps, let's say you guys create the new environment, right? We create the S tree bucket, right? And the clone command, is this a once command or we have to run in per asset? Or it's just one thing and it runs the whole night, moving the files? How could the clone works? 

6:24 | Danny
Yeah. I believe that the clone operation is, you know, can cover everything at once. It's not like you have to do it on an individual asset basis. Obviously with the, once the environment is set up, there are other dependencies in terms of making sure that we have the required access to the bucket through an Iam role, Iam, user permissions, etc, the relevant permissions, get object, put, object, delete object, and so on. So the professional services team would make sure that the first of all, we'd make sure you know, what you need to set up on your side. Professional services team would then configure the environment to use that bucket as the storage back ends. And then typically, the delivery routing and the actual CDN configuration is going to largely remain unchanged except in a situation where the customer is going to put the CDN in front, which I don't believe is the case here. Is that correct? No, correct. 

7:20 | Edgar
We're going to continue using Cloudinary CDN. 

7:24 | Danny
Cool. Yeah. So that's very straightforward. I know the other question the main big question you had was around making like the paths and making sure that those, you know, those didn't change. There are absolutely strategies we can use there to make sure that doesn't happen. And actually, it's easier when the customer has their own cname, because the urls, then within the delivery urls, when you have a cname, don't include the cloud name. So as you create a new environment, obviously, that's a new cloud, it has a new name. So when you have your own, the customer has their own cname, that makes that process a lot easier. But as I said, there's different URL preservation strategies that we can look at as part of the migration plan depending on, you know, the overall configuration. 

8:17 | Edgar
All right. So coming back. 

8:20 | Ahmed
Just a question on the new instance. So, Danny, just to summarize for byob, you definitely recommend creating a new instance that's the best way forward, not like reconfigure. Yeah. 

8:34 | Danny
Absolutely. Yeah, absolutely. You will have to have a new environment because we need to start from scratch. We need to configure the environment. Obviously byob is not that common. I'll say that up front. So, this is probably only the second time in my four years that I've actually sort of encountered this request. So, yeah, it's a new environment but within the same account. But obviously, the environments in the account are entirely segregated. Hence the need to then perform that kind of clone operation to actually copy the assets across. And by the way, I will say at this point, I know you've received certain information from support. Actually, it may be professional services is, you know, yes, you could use the cli to do this and that is a common approach when it comes to migration, but they may have other tools at their disposal, for example, some kind of scripted solution. These are kind of all the things that we need to really consider so that we can provide the most efficient approach to get from point a to point B with minimal disruption, ideally, zero disruption. Yeah, and then keep the customer happy. 

9:44 | Edgar
Right. Go ahead, Sam, Ahmed. You have another question. Fine. Cool. 

9:50 | Danny
So, 

9:52 | Edgar
all right. Let me take it back. You guys create the environment. You set it up. We create the S tree bucket. You set it up just to make sure everything is replicated in terms of configurations, right? We will provide you am access to the Uta's S tree bucket. So. 

10:11 | Danny
You can see. 

10:13 | Edgar
It now, there are options and, yes, we saw the sync, the clone. We also saw the SDK uploader, which basically is like, okay, one by one asset by asset, right? Download it and upload it, right? That was another third option. We have another fourth option, which was the using Aws tools for data sync, for instance, moving all the files from one place to another. Yeah. So from all these options… the idea is to evaluate them as you say, okay, the clone option, the sync option, the one to one option with the uploader or the data sync option, right? In your mind, high level, which one you think is more suitable for this case? 

11:06 | Danny
You're not going to like my answer here, but I think it might considering my role, I don't think it'd be appropriate for me to make that recommendation. I think that's definitely something which professional services would consider based on them having a complete understanding of what the requirements are. And really that's the purpose of this conversation today so that I can capture anything that might be missing. You can answer questions. I can then include that in the request. We can set up a follow up conversation. So, sorry for being non committal, but I think it's, the right approach here given that, those guys are going to be the experts here. 

11:41 | Edgar
Perfect. Okay. I understand. So in the, you mentioned the let's just stick with the clones for now, right? You mentioned the clone, it will move the assets, right? And it's one shot cli command. So you don't need to run by asset, just one time thing. So it will move everything, right? Yeah, it will move also the metadata the. 

12:08 | Danny
Tags, tags, contextual. Yeah, everything. Yeah. The only things that may need to be set up independently, I believe are maybe some configuration options. So, for example, I think like because the clone functionality is about moving assets. Okay? So things like upload presets if you have those and maybe some other configuration items would need to be independently configured. But again, we provide guidance on that and support. But so the cli is really about copying the assets along with the tags metadata et. 

12:46 | Edgar
Cetera. So basically, it doesn't clone the assets, it clones the environment. 

12:52 | Danny
Am I right? Not strictly speaking, so as I said, it won't include configuration. So there are certain things that have to be configured independently. The cloning is really saying, okay, take this environment and copy all of the assets from it across to this environment. And then you have various parameters within the requests which you can, you know, switches which you can tell it whether you want to include the tags, the metadata, et cetera. So there's various tags there, but there are going to be some things which still need to be configured after the actual copy or the cloning of the assets. 

13:33 | Edgar
Now, Danny? 

13:35 | Francisco
I have a question. Sorry to interrupt, sure. Go ahead aside from the assets, which well, I guess they are multimedia files, audio and video, these metadata and configuration. All of those things are also inside the S3 bucket. Everything is together. There. They're just maybe text files with configuration or Json with the metadata. Yeah. 

13:59 | Danny
That's right. Yeah. So all we're really doing is swapping out, storing all of that information in Cloudinary's environment to the customer's environment. So, fundamentally, so. 

14:09 | Francisco
We. 

14:10 | Danny
will structure the data as part of the clone. But the only difference is that we're using the customer's bucket and not Cloudinary bucket for that storage. 

14:23 | Edgar
Okay. All right. And if this is one shot cli command one time, sorry, it… will be a risk of I don't know timeouts or anything because we're moving three terabytes of data. Yeah. 

14:42 | Danny
No, there shouldn't be. So there's a, I know that there's specifically an async option. You shouldn't experience any issues with it. The way that the solution is configured. Obviously, it's run through the terminal. You're not running it through a browser, for example. So I'd probably. And again, we'll double check these professional services. But from when I've looked at this before generally speaking, use the async option to clone asynchronously to avoid any potential issues. You also, but you also by the way, have I think I remember incorrectly? There's also the option of delivering web notifications to confirm cloning when the cloning is complete, and then it's been successful. 

15:24 | Edgar
Nice. That would be nice. Yeah, because one of the let's say, biggest question is the cutout right? How much time this is gonna take? Yeah, because we're thinking maybe we should move, let's say, for instance, the employees profiles pictures, right? They are in Cloudinary and that's a small data set, a small set of assets. So maybe move those test, the, their application. The one that uses those urls. Okay. It works. Now move another set of images, but with this clone, I understand it moves everything, right? 

16:04 | Danny
So you can use a search expression. So if you wanted to limit or filter the assets that are actually being cloned, you can supply a search expression. So if you want to just do a specific folder or assets which have a certain prefix or certain name or whatever it might be, you can do that. It is, it is optional, but that would give you a way, to do that. If, if that's what you want, if you wanted to do as, you know, a certain subset of assets first before considering doing the, you know, a full clone of everything. 

16:42 | Edgar
Okay. That would be great. That would be great. So in that way we can start moving data sets with these, the search expression, right? I know it's I know the answer and the answer, it will be no, but is the time, how do you, how much time do you think you will take to move all these assets? 

17:05 | Danny
Because the answer isn't the answer isn't no. The answer is, I don't know what I can do if we, how I, I'll be honest, I haven't looked to see the kind of volume we're talking about here in terms of both the number of us, obviously the number of assets and the total file size both have an impact. What, again, what I can do following this conversation and obviously I'll follow up with the team and say, look, hey based on your experience of doing these because obviously they've done a lot. I've only done like one or two maybe, you know, roughly what kind of time frame are we talking about here, to complete a clone? And then I'll just let you know we'll try to give you like a sensible range all. 

17:45 | Edgar
Right. Then you. 

17:47 | Francisco
Have one more question… you know, as an another alternative then we can put in the table is what I was discussing with Edgar, another tool called S3, a batch replication. That tool probably will help us to, for the cut over basically to forget about it because we can configure that and mark in the origin bucket. A live replication rule will always gonna be there. So we can do it once the initial one, the full three terabytes, and then afterwards, if there any new object or asset uploaded there, it will automatically get replicated into the new bucket. But to evaluate if we can do that, I need some information on how the bucket is configured. So those questions, can we maybe forward them to the professional services team? Yeah? 

18:46 | Danny
I mean, in the first instance, I would say because I'll essentially act as a liaison at this point, I know you've been passed from support and I'm now engaging with you directly. So I would say I will follow up after this to everyone on the call, use that thread to share questions. Because what we need to do is then build a very clear picture of the requirements to, so we can submit that into the professional services team for evaluation and questions like this, which, I, unfortunately, I can't answer they can do. And the goal will then to be to have a follow up conversation to actually, you know, go beyond this kind of the theoreticals, and the options and actually formulate, a defined plan. 

19:31 | Francisco
Okay. Then, yep, we're gonna put together all of these and questions sounds. 

19:39 | Edgar
Good. Right now. Another question. In the beginning, we started having these sessions with question and answers. They initially recommended the sync cli command was, and we're trying to look at the documentation but it wasn't clear what is the difference between sync and clone? Do you know by any chance? Yeah. 

20:01 | Danny
So cloning allows you to copy between in directly between environments, whereas the sync option will synchronize between a local folder and a Cloudinary folder. So it's more, yeah. So that, that's the fundamental difference all. 

20:20 | Edgar
Right. So let's think now. Okay. So we're local to clothing now. Yeah. Okay. Good. Now, we haven't seen the Uta's Cloudinary bucket. We don't know how it's structured. 

20:36 | Danny
Yeah. 

20:38 | Edgar
Now, does it off? You skip when you let's start from the beginning? The user journey, the Guy uploads a file through apis because that's what they do. Do, you know if the file is obfuscated or the folder has any structure or it's just a bucket with everything in the same route. So. 

21:01 | Danny
The dam really you could think of is operating in the same way as any other file system. So the customer can generate have their own folder structure. And then obviously, you know, that could be a nested folder structure. And then when they upload assets into that folder structure, they have the option at the time of upload or through upload configuration options to determine how the file should be named when it's uploaded. So do we create a random file name to completely obfuscate the file name? Or do they want to use the actual file name of the asset being uploaded? So that's the customer's choice. And obviously, that will then determine what the delivery URL looks like. Now, of course, there are security options, signed urls, etc. So if they need to add security to prevent unwanted people accessing the assets, then that's also available to them as well. Again, I haven't looked into the details of exactly how they're using the platform at the moment. I am in there right now, so I can probably just look and give you some kind of indication of how they're set up. Yeah. So they have a folder structure in place. It doesn't look like it's particularly heavily nested. And let me just look at the assets. So they are using, by the looks of it, they are using the random file name option. So when I look at the file names, there's nothing meaningful there. So they're using the option to just assign a random around 20 characters. I think the file names are certainly at least in a lot of cases, I am seeing a few assets which do have some form of meaningful name. So it might be that they have different upload presets configured for different purposes. And in some cases, the upload preset is just… giving these random file names, but maybe in other circumstances, they're using an upload preset which is taking the actual file name and applying that. So again, in fact, if I look at the, let me just quickly take a look at their upload. Yeah. So they've got a bunch of upload presets configured. So it's likely they have different presets for different purposes that might be processing the uploaded files in different ways. 

23:23 | Edgar
Okay. So they're most likely as you said, using the defaults plus some changes here and there. Yeah, this is not a top priority. However I need to ask they in business from the business area, they would like to have some meaningful file name and probably a folder structure. Is that possible with the clone or that's yeah. 

23:49 | Danny
I mean, the intention of the clone would be to replicate what they have today, so. 

23:54 | Edgar
It will replicate obfuscate it as it is but not. 

23:58 | Danny
Meaningful. Yeah. My anticipation would be that the structure, the file names et cetera, would be the same. Okay? So all. 

24:10 | Edgar
Right. So let's say we copy everything as of today in the same folder names, whatever? 

24:16 | Danny
Yeah. Is there? 

24:18 | Edgar
A way of changing that afterwards? Let's say after migration, we migrate all the three terabytes, everybody's happy. It's working perfect. Now, we want to go through the files and, you know, make the more human friendly. Is there a way to do that? 

24:38 | Danny
Yeah. So, and again, let me just quickly take a look at the current configuration. So their main environment, okay. Good. That's good. So their main environment is configured in what we call dynamic folders mode. So I'll try to explain this. Historically Cloudinary had one folder mode which was fixed folders. And what that meant was you could build out a folder hierarchy however you pleased, and you would upload your assets into that folder hierarchy. But there was only ever one id, associated assets, which was called the public id. The challenge with that approach is that if I moved one of my assets from folder a to folder B or I renamed the asset, it would break the delivery URL. With the introduction of dynamic folders. It essentially decoupled the… file's folder location from the delivery URL, meaning that I can now move assets from one folder to another without affecting the delivery URL and therefore breaking things downstream. Likewise. There's now effectively a secondary id for files for each asset. So what was the public id? The one and only id previously that's now kind of hidden, that generally stays consistent because that ensures then that we don't break any urls, but the customer can go in and they can rename assets to something else without breaking anything else. So if they wanted to take one of those random ids that I mentioned a moment ago and give it a meaningful name that's not going to break the delivery URL. Yeah. So just, and by the way, some customers still prefer to use fixed folders mode, it's a customer's choice. Each environment the customer has can be configured in either way. But in this case, I can tell you that their production environment is using the dynamic folders. 

26:50 | Edgar
So there's a possibility, but that will be afterwards as another phase two of the project like make this. 

26:57 | Danny
Yeah, potentially you could go through, you could extract the information about the assets and then make edits and then you maybe use the, or use our apis and then go through and rename assets or do whatever you want. Obviously the API, we are an API first solution. So anything you can do inside the platform, you can absolutely do through the apis and typically more efficiently more quickly. But again, as and when we know what the actual need is from the customer, we can provide the best practice advice on which option to use. 

27:33 | Edgar
Got it. All right. Now. We're short of time. We may need to schedule something later, however, I think I got a very good idea of the process. Yeah. 

27:44 | Danny
We. 

27:45 | Edgar
do have questions about the CDN. So the CDN catches the file and we're creating a new environment. We copy the files, however the URL will remain with the cname and DNS. It will remain pretty much the same, right? But does the cache also need to be refreshed? It will take some time or is it also? Yeah. 

28:11 | Danny
It's yeah. So, I think that might largely depend on what we spoke about earlier. We talked about if the customer has the consistent cname, and we just effectively move that cname over to the new environment. And everything else is the same. We might not need to do anything in terms of purging or, you know, cache warm up. But again, I think those things are probably better directed to professional services so they can give you a more accurate overview. One thing I just want to call out really quickly because I think I don't think I mentioned when we were talking about earlier about the clone feature. I said about the metadata specifically, it includes tags and… contextual metadata. However, I noticed the customer does have a structured metadata taxonomy now that is not moved as part of the clone, but we can put a process in place to replicate it and make sure it's updated. Just wanted to call that out just to make sure I wasn't misleading you, but just know that it can be managed as part of the process. 

29:15 | Edgar
And, okay. So we will need to deep dive into that one. Yeah, because they will say, they will probably say, okay, this is incomplete. We need to move everything. 

29:26 | Danny
Yeah. That's right? So it will be a stage process. The clone is going to fundamentally move. So the clone can fundamentally move the actual binary files along with any attached contextual metadata. Any tags. We've already mentioned, the fact that there will be some things that have to be reconfigured like upload presets and then structured metadata. We have processes in place to handle that. So, for example, you can do a bulk metadata export from the existing environment into a file, move the file, you know, clone the files across and then use that bulk export that you've already created to re import and attach that metadata back to the, you know, back to the files in the new environment. So it's not difficult. You know, obviously, we go through these migration processes all the time, not necessarily in this particular way with the environment to environment that's maybe less common. But obviously, we are bringing on board new customers all the time. And so migration processes are fundamental to our onboarding process. And so a lot of the things I'm talking about here are similar to what we would go through with a new customer who maybe has assets in a completely different source, right? So that's why we have all of these processes available to make sure that everything is in place. 

30:46 | Edgar
All right. Awesome. Okay. We are on the hour. Thank you very much. Sure. 

30:52 | Leo
I'm sorry, just one quick one. What happens with the new instance when we're done with it? Do you just delete it? 

31:00 | Danny
You mean the current one? The old? 

31:02 | Edgar
Instance? 

31:04 | Danny
Yeah, absolutely. So, I mean even the customer can actually do it themselves if they're confident they can go into their product environments and hit the delete? I. 

31:14 | Leo
Guess I'm asking, is it theirs, do they own it? Can they do with it? What they want? Can they keep it if they wanted to? 

31:19 | Danny
Yeah, of course, they can. The only thing to bear in mind then is that if they keep it, then they are utilizing storage, they probably won't be utilizing delivery anymore because they'll be delivering from the new environment. But any storage that's there is going to contribute to their storage consumption and the units that they use from Cloudinary. So, yeah, I mean, that would be a decision on them if they want to keep it sure, but maybe they do for a time until they're satisfied that everything's okay. But after that, I would suggest delete it to free up that storage. 

31:52 | Leo
Okay. And I guess I may want to be sure to ask how we follow up with professional services that's going? 

31:59 | Danny
To be on me, okay, we do need to formalize an arrangement with them. So there is a proper process we have to go through to request their time. What will typically then happen will be an initial engagement where we'll go through scoping briefing. They'll have access to everything we've discussed here. So they'll be briefed ahead of time, but that will then be about understanding what's involved. And also potentially, you know, here we're looking at potential issues depending on their level of involvement. There's likely to be a cost involved. So they typically have like an hourly arrangement or maybe even some kind of lightweight implementation package. But again, I don't want to speculate on that. That's going to be for them to determine based on how we progress through the conversation. 

32:47 | Edgar
Okay. Thank you. Sure. I feel. 

32:52 | Danny
like I'm surprised how quickly the time went but it's very nice to meet you all nice. 

32:57 | Leo
To meet. 

32:57 | Danny
You too. I will. Yeah, I pulled up some of the questions. Other questions I did fill those in. I'll ping them over to you. So got those off. I'll follow up with the summary call recording and like I said, we'll initiate that engagement with professional services so we can get the ball rolling and on a bit more of a formal level and hopefully set up that initial sort of initial scoping session as soon as possible. 

33:27 | Edgar
Okay. All right. Yeah. I think the next thing is to set up this meeting with professional services and also, you know, gather this with Uta. Okay. This is how we're going and these are the possible options. Sure thing. Sounds good. Thank you very much. No worries. 

33:44 | Danny
It's nice to meet you all. Take care. See you later. Bye bye. Have a nice day. 

33:48 | Edgar
Same bye. 