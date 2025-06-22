-- users 
{
  _id: ObjectId,
  name: String,
  email: String,
  rollNumber: String,
  createdAt: Date
}
-- codekata
{
  _id: ObjectId,
  userId: ObjectId, 
  problemId: String,
  problemName: String,
  solvedAt: Date,
  status: String 
}

-- attendance
{
  _id: ObjectId,
  userId: ObjectId, 
  date: Date,
  status: String /
}
-- topics

{
  _id: ObjectId,
  topicName: String,
  taughtDate: Date,
  courseId: String
}
-- tasks
{
  _id: ObjectId,
  userId: ObjectId, 
  topicId: ObjectId, 
  taskName: String,
  assignedDate: Date,
  submittedDate: Date,
  status: String 
}
-- company_drives
{
  _id: ObjectId,
  companyName: String,
  driveDate: Date,
  participants: [ObjectId] 
}

-- mentors
{
  _id: ObjectId,
  name: String,
  email: String,
  mentees: [ObjectId] 
}
-- Find all the topics and tasks taught in the month of October (any year)

db.topics.find({
  taughtDate: {
    $gte: ISODate("2020-10-01T00:00:00Z"),
    $lt: ISODate("2020-11-01T00:00:00Z")
  }
}).toArray();


db.tasks.find({
  assignedDate: {
    $gte: ISODate("2020-10-01T00:00:00Z"),
    $lt: ISODate("2020-11-01T00:00:00Z")
  }
}).toArray();

-- Find all the company drives which appeared between 15-Oct-2020 and 31-Oct-2020
db.company_drives.find({
  driveDate: {
    $gte: ISODate("2020-10-15T00:00:00Z"),
    $lte: ISODate("2020-10-31T23:59:59Z")
  }
}).toArray();

-- Find all the company drives and students who appeared for the placement
db.company_drives.aggregate([
  {
    $lookup: {
      from: "users",
      localField: "participants",
      foreignField: "_id",
      as: "students"
    }
  },
  {
    $project: {
      companyName: 1,
      driveDate: 1,
      "students.name": 1,
      "students.email": 1,
      "students.rollNumber": 1
    }
  }
]).toArray();
-- Find the number of problems solved by the user in codekata
db.codekata.aggregate([
  {
    $match: { status: "solved" }
  },
  {
    $group: {
      _id: "$userId",
      problemCount: { $sum: 1 }
    }
  },
  {
    $lookup: {
      from: "users",
      localField: "_id",
      foreignField: "_id",
      as: "user"
    }
  },
  {
    $unwind: "$user"
  },
  {
    $project: {
      userName: "$user.name",
      problemCount: 1
    }
  }
]).toArray();
-- Find all the mentors with mentee count more than 15
db.mentors.find({
  mentees: { $size: { $gt: 15 } }
}).toArray();

-- Find the number of users who are absent and task not submitted between 15-Oct-2020 and 31-Oct-2020
db.attendance.aggregate([
  {
    $match: {
      date: {
        $gte: ISODate("2020-10-15T00:00:00Z"),
        $lte: ISODate("2020-10-31T23:59:59Z")
      },
      status: "absent"
    }
  },
  {
    $group: {
      _id: "$userId",
      absentDates: { $push: "$date" }
    }
  },
  {
    $lookup: {
      from: "tasks",
      let: { userId: "$_id", absentDates: "$absentDates" },
      pipeline: [
        {
          $match: {
            $expr: {
              $and: [
                { $eq: ["$userId", "$$userId"] },
                { $gte: ["$assignedDate", ISODate("2020-10-15T00:00:00Z")] },
                { $lte: ["$assignedDate", ISODate("2020-10-31T23:59:59Z")] },
                { $eq: ["$status", "not_submitted"] }
              ]
            }
          }
        }
      ],
      as: "tasks"
    }
  },
  {
    $match: {
      tasks: { $ne: [] }
    }
  },
  {
    $lookup: {
      from: "users",
      localField: "_id",
      foreignField: "_id",
      as: "user"
    }
  },
  {
    $unwind: "$user"
  },
  {
    $project: {
      userName: "$user.name",
      absentDates: 1,
      taskCount: { $size: "$tasks" }
    }
  },
  {
    $count: "totalUsers"
  }
]).toArray();
